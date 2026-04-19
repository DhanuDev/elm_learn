# IIS Connection Reset Error (10054) - Production Fix Guide
**Generated:** 2026-04-19 | **Error Pattern:** Socket closed by remote host | **Severity:** HIGH

---

## EXECUTIVE SUMMARY

Your `elmbpmservice` API is forcibly closing TCP connections. This is **not** an IIS misconfiguration — it's the remote server rejecting or timing out your requests. Your application layer must handle this gracefully.

**Impact:** 41 connection errors in 10 minutes → ~4 errors/min → affects customer task submissions and process cancellations

**Fix Strategy:** (1) Diagnose remote API health, (2) Add resilience in code, (3) Tune IIS for better timeout handling

---

## SECTION 1: IMMEDIATE DIAGNOSTICS (0-5 MIN)

### 1.1 Check Remote API Connectivity

```powershell
# PowerShell as Administrator

# Test if the remote server is reachable
ping hhs.sbis.hrsd.gov.sa
Test-NetConnection -ComputerName "hhs.sbis.hrsd.gov.sa" -Port 443 -InformationLevel Detailed

# Check current HTTP connections from your server
netstat -ano | findstr "ESTABLISHED" | findstr ":443"
netstat -ano | findstr "ESTABLISHED" | findstr ":80"

# Count connections to the problematic API
netstat -ano | findstr "hhs.sbis.hrsd.gov.sa"

# If too many connections, they're piling up → timeout issue
# If few connections, likely rejected/rate-limited
```

### 1.2 Check IIS Application Pool Health

```powershell
# Check app pool state
appcmd list apppool

# Look for recycling logs
Get-Content "C:\Windows\System32\inetsrv\config\applicationhost.config" | findstr "recycling"

# Check if app pool is recycling frequently
# In Event Viewer: Windows Logs → Application → Search for "W3SVC" errors
Get-EventLog -LogName "Application" -Source "W3SVC" -Newest 20
```

### 1.3 Check System Resource Pressure

```powershell
# CPU Usage
Get-WmiObject win32_processor | Select-Object LoadPercentage

# Memory Usage
Get-WmiObject win32_operatingsystem | Select-Object TotalVisibleMemorySize, FreePhysicalMemory

# If either CPU >75% or Memory >85% FREE, you have resource contention
```

### 1.4 Enable IIS Failed Request Tracing

This is **critical** for seeing exactly where connections fail:

```powershell
# Open IIS Manager
# 1. Select your application
# 2. Right-click → "Edit Feature Permissions"
# 3. Check "Read" and "Execute"
# 4. Go to "Failed Request Tracing Rules"
# 5. Add rule:
#    - URL: *.aspx (or your extension)
#    - Status codes: 500-599
#    - Module: All
# 6. Enable: Trace Failed Requests
# 7. Logs saved in: %SystemRoot%\System32\logfiles\FailedReqLogFiles\
```

---

## SECTION 2: CODE-LEVEL FIXES (DEPLOY IMMEDIATELY)

Your error trace shows `Refit.ApiException: 500 (Internal Server Error)` — the remote server *is responding* but then closing. Implement this:

### 2.1 Add Timeout & Retry Logic (C# / .NET)

```csharp
// In your Refit interface or HTTP client configuration

using Polly;
using Polly.CircuitBreaker;
using Refit;

// Define a resilience policy
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .Or<OperationCanceledException>()
    .OrResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
    .WaitAndRetryAsync(
        retryCount: 3,
        sleepDurationProvider: attempt => 
            TimeSpan.FromSeconds(Math.Pow(2, attempt)), // Exponential backoff: 1s, 2s, 4s
        onRetry: (outcome, delay, retryCount, context) =>
        {
            _logger.LogWarning(
                $"Retry {retryCount} after {delay.TotalSeconds}s due to: {outcome.Exception?.Message}");
        }
    );

var circuitBreakerPolicy = Policy
    .Handle<HttpRequestException>()
    .OrResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode && (int)r.StatusCode >= 500)
    .CircuitBreakerAsync(
        handledEventsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromSeconds(30),
        onBreak: (outcome, duration) =>
        {
            _logger.LogError($"Circuit breaker opened for {duration.TotalSeconds}s");
        }
    );

// Combine policies
var combinedPolicy = Policy.WrapAsync(retryPolicy, circuitBreakerPolicy);

// Apply to your HTTP client
var httpClientHandler = new HttpClientHandler
{
    MaxConnectionsPerServer = 10,
    AutomaticDecompression = System.Net.DecompressionMethods.GZip | 
                             System.Net.DecompressionMethods.Deflate
};

var httpClient = new HttpClient(httpClientHandler)
{
    Timeout = TimeSpan.FromSeconds(30) // 30s timeout
};

var refitClient = RestService.For<IElmBpmService>(
    httpClient,
    new RefitSettings
    {
        ContentSerializationHandler = new JsonContentSerializerHandler()
    }
);

// Wrap API calls
try
{
    var result = await combinedPolicy.ExecuteAsync(async () =>
        await refitClient.GetTasks(request)
    );
}
catch (BrokenCircuitException)
{
    _logger.LogError("Service unavailable - circuit breaker active");
    // Return cached result or user-friendly error
}
catch (HttpRequestException ex) when (ex.InnerException is OperationCanceledException)
{
    _logger.LogError("Request timeout to external API");
    // Retry with longer timeout or fallback
}
```

### 2.2 Reduce Concurrent Requests to Remote API

```csharp
// Use a SemaphoreSlim to limit concurrent calls
private static readonly SemaphoreSlim _apiThrottle = 
    new SemaphoreSlim(5, 5); // Max 5 concurrent requests

public async Task<TaskResponse> GetTasksSafely(string userId)
{
    using (await _apiThrottle.WaitAsync())
    {
        try
        {
            return await _refitClient.GetTasks(new { userId });
        }
        catch (HttpRequestException ex)
        {
            // Log and handle gracefully
            _logger.LogWarning($"External API error: {ex.Message}");
            return new TaskResponse { IsSuccess = false };
        }
    }
}
```

### 2.3 Increase .NET TCP Connection Limits

Add this to `appsettings.json` or startup code:

```csharp
// In Program.cs or Startup.cs
ServicePointManager.DefaultConnectionLimit = 10;
ServicePointManager.MaxServicePointIdleTime = 60000; // 60 seconds

// Enable TCP keep-alive for long-lived connections
var handler = new SocketsHttpHandler
{
    PooledConnectionIdleTimeout = TimeSpan.FromSeconds(60),
    PooledConnectionLifetime = TimeSpan.FromMinutes(2),
    AutomaticDecompression = DecompressionMethods.All
};
```

---

## SECTION 3: IIS CONFIGURATION CHANGES

### 3.1 Increase App Pool Timeouts

```powershell
# PowerShell as Administrator

# 1. Increase Shutdown Time Limit (default: 90 sec)
appcmd set apppool "DefaultAppPool" -processModel.shutdownTimeLimit:300

# 2. Increase Idle Timeout (default: 20 min)
appcmd set apppool "DefaultAppPool" -processModel.idleTimeout:00:30:00

# 3. Set Regular Time Interval for recycling (less aggressive)
appcmd set apppool "DefaultAppPool" -recycling.periodicRestart.time:00:00:00

# 4. Queue Length (buffer for pending requests)
appcmd set apppool "DefaultAppPool" -queueLength:2000
```

### 3.2 Increase Request Timeout in IIS

```powershell
# Default: 120 seconds (2 min)
# If your external API takes >2 min, requests will timeout

# Check current setting
Get-WebConfigurationProperty -Filter "system.webServer/requestFiltering" `
    -Name "requestTimeout" `
    -PSPath "IIS:\Sites\YourSiteName"

# Increase to 10 minutes for long-running operations
Set-WebConfigurationProperty -Filter "system.webServer/requestFiltering" `
    -Name "requestTimeout" `
    -Value "00:10:00" `
    -PSPath "IIS:\Sites\YourSiteName"
```

### 3.3 Enable Connection Keep-Alive

```powershell
# In web.config, add to <system.webServer>:
$webConfigPath = "C:\inetpub\wwwroot\YourApp\web.config"

# Or via PowerShell:
Set-WebConfigurationProperty -Filter "system.webServer/httpProtocol" `
    -Name "keepAlive" `
    -Value "true" `
    -PSPath "IIS:\Sites\YourSiteName"
```

### 3.4 Disable HTTP/2 (Possible Cause of Resets)

```powershell
# HTTP/2 can cause connection resets in some .NET versions
# In web.config, modify <system.webServer>:

# <security>
#   <requestFiltering>
#     <verbs>
#       <add verb="GET" allowed="true" />
#       <add verb="POST" allowed="true" />
#     </verbs>
#   </requestFiltering>
# </security>

# Disable HTTP/2 in IIS:
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\HTTP\Parameters" `
    /v "EnableHttp2" /t REG_DWORD /d 0 /f

# Restart IIS
iisreset /restart
```

---

## SECTION 4: NETWORK & FIREWALL CHECKS

### 4.1 Check Proxy/Firewall Idle Connection Timeout

```powershell
# Some network devices disconnect idle HTTP connections after 30-60 seconds
# Test if connection stays open:

# Create a simple keep-alive mechanism in your code:
var handler = new HttpClientHandler();
var httpClient = new HttpClient(handler)
{
    Timeout = TimeSpan.FromSeconds(60)
};

# Or use: Connection: keep-alive headers
var client = new HttpClient();
client.DefaultRequestHeaders.Connection.Add("keep-alive");
client.DefaultRequestHeaders.Add("Keep-Alive", "timeout=30, max=100");
```

### 4.2 Verify No Connection Limits from ISP/Proxy

```powershell
# Check Windows Firewall connection tracking
netsh advfirewall show allprofiles | findstr "State"

# Temporarily disable to test (NOT for production)
netsh advfirewall set allprofiles state off

# Re-enable after testing
netsh advfirewall set allprofiles state on

# Check for proxy rules
netsh winhttp show proxy
```

---

## SECTION 5: MONITORING & ALERTING

### 5.1 Create Custom Perfmon Counters

```powershell
# Monitor HTTP connection failures in real-time

# Create counter for tracking:
# 1. HTTP Requests / sec (should be stable)
# 2. Connection Timeouts (should be 0 or very low)
# 3. App Pool Recycles (should not happen frequently)

# View in Perfmon (perfmon.msc):
# System → TCP Connections → Connections Reset
# .NET CLR Networking → Connections Opened / Closed
```

### 5.2 Add Application-Level Logging

```csharp
// Log all external API calls with detailed info
public class ApiCallInterceptor : DelegatingHandler
{
    private readonly ILogger<ApiCallInterceptor> _logger;
    
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken cancellationToken)
    {
        var stopwatch = Stopwatch.StartNew();
        try
        {
            var response = await base.SendAsync(request, cancellationToken);
            stopwatch.Stop();
            
            _logger.LogInformation(
                "API Call: {Method} {Uri} → {StatusCode} in {ElapsedMs}ms",
                request.Method, request.RequestUri, response.StatusCode, stopwatch.ElapsedMilliseconds);
            
            return response;
        }
        catch (HttpRequestException ex)
        {
            stopwatch.Stop();
            _logger.LogError(ex,
                "API Call Failed: {Method} {Uri} after {ElapsedMs}ms - {Message}",
                request.Method, request.RequestUri, stopwatch.ElapsedMilliseconds, ex.Message);
            throw;
        }
        catch (OperationCanceledException ex)
        {
            stopwatch.Stop();
            _logger.LogError(ex,
                "API Call Timeout: {Method} {Uri} after {ElapsedMs}ms",
                request.Method, request.RequestUri, stopwatch.ElapsedMilliseconds);
            throw;
        }
    }
}

// Register in DI:
services.AddHttpClient<IElmBpmService>()
    .AddHttpMessageHandler<ApiCallInterceptor>();
```

---

## SECTION 6: VALIDATION CHECKLIST

- [ ] **Ran netstat check** — confirmed if connection pool is growing
- [ ] **Checked Event Viewer** — captured error messages with timestamps
- [ ] **Verified remote API is online** — `ping` and `telnet` tests passed
- [ ] **Deployed retry logic** — exponential backoff + circuit breaker in place
- [ ] **Increased app pool timeout** — to at least 300 seconds
- [ ] **Added semaphore throttle** — limiting concurrent calls to ≤5
- [ ] **Enabled HTTP Keep-Alive** — to prevent idle disconnects
- [ ] **Disabled HTTP/2** — if using .NET Framework <4.7.2
- [ ] **Created monitoring rules** — in Perfmon or Application Insights
- [ ] **Tested with load** — confirmed errors are reduced

---

## SECTION 7: ESCALATION PATH

**If errors continue after above steps:**

1. **Contact remote API owner** — they may be rate-limiting or restarting
   - Ask for: connection limits, timeout policy, IP whitelist requirements
   
2. **Capture HTTP Archive (HAR) file**
   - Use Fiddler or browser DevTools to record full request/response
   - Include timestamps, headers, and payload

3. **Enable IIS ISAPI Tracing**
   - Run: `%SystemRoot%\System32\inetsrv\appcmd set config /section:system.webServer/tracing /enabled:true`
   - Logs to: `%SystemRoot%\System32\LogFiles\HTTPERR\`

4. **Engage Network Team**
   - Check for proxy/firewall rules terminating idle connections
   - Ask for connection timeout policy on your network segment

---

## QUICK REFERENCE: Commands

```powershell
# All-in-one diagnostic script
Write-Host "=== IIS Connection Error Diagnostics ===" -ForegroundColor Green
Write-Host "`n[1] Checking remote API connectivity..."
Test-NetConnection -ComputerName "hhs.sbis.hrsd.gov.sa" -Port 443

Write-Host "`n[2] Current ESTABLISHED connections..."
netstat -ano | findstr "ESTABLISHED" | wc -l

Write-Host "`n[3] App pool state..."
appcmd list apppool

Write-Host "`n[4] Recent IIS errors (last hour)..."
Get-EventLog -LogName "Application" -Source "W3SVC" -After (Get-Date).AddHours(-1) | `
    Select-Object TimeGenerated, Message | Head -5

Write-Host "`n[5] System resources..."
Get-WmiObject win32_processor | Select-Object LoadPercentage
Get-WmiObject win32_operatingsystem | Select-Object `
    @{Name="MemoryUsage%";Expression={[math]::Round(100 * $_.TotalVisibleMemorySize - $_.FreePhysicalMemory / $_.TotalVisibleMemorySize, 2)}}
```

---

**Last Updated:** 2026-04-19T09:03  
**Status:** Active Investigation  
**Next Review:** After deploying code changes
