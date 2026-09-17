# FamilyGuard Instrumentation Tests

## Overview
Instrumentation tests for Android services that require system integration, including AccessibilityService, NotificationListenerService, VpnService, LocationService, and Device Admin functionality.

## Test Environment Setup

### 1. Test Device Configuration
```kotlin
@get:Rule
val testRule = ServiceTestRule()

@get:Rule
val grantPermissionRule = GrantPermissionRule.grant(
    Manifest.permission.ACCESS_FINE_LOCATION,
    Manifest.permission.ACCESS_BACKGROUND_LOCATION,
    Manifest.permission.RECORD_AUDIO,
    Manifest.permission.CAMERA
)

@Before
fun setup() {
    // Enable test mode
    TestMode.enable()
    
    // Grant runtime permissions
    InstrumentationRegistry.getInstrumentation().uiAutomation
        .executeShellCommand("pm grant ${ContextUtils.getTargetPackage()} " +
            "android.permission.PACKAGE_USAGE_STATS")
    
    // Set up test data
    TestDataRepository.initialize()
}
```

### 2. Service Test Base Class
```kotlin
abstract class BaseServiceTest {
    
    protected lateinit var context: Context
    protected lateinit var serviceController: ServiceController
    
    @Before
    fun setupServiceTest() {
        context = InstrumentationRegistry.getInstrumentation().targetContext
        serviceController = ServiceController(context)
        
        // Clear previous test data
        clearTestData()
        
        // Initialize test database
        initializeTestDatabase()
    }
    
    @After
    fun cleanupServiceTest() {
        // Stop all services
        serviceController.stopAllServices()
        
        // Clean up test data
        cleanupTestData()
        
        // Reset test mode
        TestMode.reset()
    }
    
    protected fun startService(serviceClass: Class<*>): IBinder {
        val intent = Intent(context, serviceClass)
        return testRule.startService(intent)
    }
}
```

## Accessibility Service Tests

### 1. Social Media Text Extraction Tests
```kotlin
@RunWith(AndroidJUnit4::class)
class AccessibilityServiceTest : BaseServiceTest() {
    
    private lateinit var accessibilityService: FamilyGuardAccessibilityService
    
    @Test
    fun testWhatsAppMessageExtraction() {
        // Given
        accessibilityService = startService(
            FamilyGuardAccessibilityService::class.java
        ) as FamilyGuardAccessibilityService
        
        val testPackage = "com.whatsapp"
        val testMessage = "Hello from test"
        
        // When - Simulate WhatsApp UI
        simulateWhatsAppChat(testPackage, testMessage)
        
        // Then - Verify extraction
        val extractedMessages = accessibilityService.getExtractedMessages()
        assertThat(extractedMessages).hasSize(1)
        assertThat(extractedMessages[0].messageText).contains(testMessage)
        assertThat(extractedMessages[0].appPackage).isEqualTo(testPackage)
    }
    
    @Test
    fun testInstagramDMExtraction() {
        // Given
        accessibilityService = startService(
            FamilyGuardAccessibilityService::class.java
        ) as FamilyGuardAccessibilityService
        
        // When - Simulate Instagram DM
        simulateInstagramDM("com.instagram.android", "Test DM content")
        
        // Then
        val socialCaptures = SocialCaptureRepository.getRecentCaptures()
        assertThat(socialCaptures).isNotEmpty()
        assertThat(socialCaptures[0].appName).contains("Instagram")
    }
    
    @Test
    fun testKeyloggingFromEditText() {
        // Given
        accessibilityService = startService(
            FamilyGuardAccessibilityService::class.java
        ) as FamilyGuardAccessibilityService
        
        // When - Simulate text input
        simulateTextInput("com.instagram.android", "search_query", "test search")
        
        // Then
        val keylogEvents = KeylogRepository.getRecentEvents()
        assertThat(keylogEvents).hasSize(1)
        assertThat(keylogEvents[0].inputText).isEqualTo("test search")
        assertThat(keylogEvents[0].appPackage).isEqualTo("com.instagram.android")
    }
    
    @Test
    fun testSensitiveFieldDetection() {
        // Given
        accessibilityService = startService(
            FamilyGuardAccessibilityService::class.java
        ) as FamilyGuardAccessibilityService
        
        // When - Simulate password input
        simulatePasswordInput("com.facebook.katana", "password123")
        
        // Then
        val keylogEvents = KeylogRepository.getRecentEvents()
        assertThat(keylogEvents).hasSize(1)
        assertThat(keylogEvents[0].isSensitive).isTrue()
        assertThat(keylogEvents[0].inputType).isEqualTo("password")
    }
    
    private fun simulateWhatsAppChat(packageName: String, message: String) {
        // Create mock AccessibilityNodeInfo for WhatsApp chat
        val rootNode = createMockAccessibilityNode(packageName)
        val chatNode = createChatNode(message)
        
        // Simulate accessibility events
        val event = AccessibilityEvent.obtain()
        event.eventType = AccessibilityEvent.TYPE_VIEW_TEXT_CHANGED
        event.packageName = packageName
        event.source = chatNode
        
        accessibilityService.onAccessibilityEvent(event)
    }
}
```

### 2. Service Lifecycle Tests
```kotlin
@Test
fun testServiceStartStopCycle() {
    // Given
    val serviceController = ServiceController(context)
    
    // When - Start service
    serviceController.startAccessibilityService()
    
    // Then - Verify service is running
    assertThat(serviceController.isAccessibilityServiceRunning()).isTrue()
    
    // When - Stop service
    serviceController.stopAccessibilityService()
    
    // Then - Verify service stopped
    assertThat(serviceController.isAccessibilityServiceRunning()).isFalse()
}

@Test
fun testServicePersistenceAfterReboot() {
    // Given
    serviceController.startAccessibilityService()
    
    // When - Simulate device reboot
    simulateDeviceReboot()
    
    // Then - Service should auto-restart
    eventually(5, TimeUnit.SECONDS) {
        assertThat(serviceController.isAccessibilityServiceRunning()).isTrue()
    }
}
```

## Notification Service Tests

### 1. Notification Capture Tests
```kotlin
@RunWith(AndroidJUnit4::class)
class NotificationServiceTest : BaseServiceTest() {
    
    private lateinit var notificationService: FamilyGuardNotificationService
    
    @Test
    fun testNotificationCapture() {
        // Given
        notificationService = startService(
            FamilyGuardNotificationService::class.java
        ) as FamilyGuardNotificationService
        
        val testNotification = createTestNotification(
            packageName = "com.whatsapp",
            title = "New Message",
            text = "John: Hello there",
            timestamp = System.currentTimeMillis()
        )
        
        // When - Post notification
        postNotification(testNotification)
        
        // Then - Verify capture
        val notificationLogs = NotificationRepository.getRecentLogs()
        assertThat(notificationLogs).hasSize(1)
        assertThat(notificationLogs[0].title).isEqualTo("New Message")
        assertThat(notificationLogs[0].text).contains("Hello there")
    }
    
    @Test
    fun testNotificationGroupHandling() {
        // Given
        notificationService = startService(
            FamilyGuardNotificationService::class.java
        ) as FamilyGuardNotificationService
        
        // When - Post grouped notifications
        postNotificationGroup("com.instagram.android", listOf(
            createTestNotification(title = "Like 1"),
            createTestNotification(title = "Like 2"),
            createTestNotification(title = "Like 3")
        ))
        
        // Then
        val notificationLogs = NotificationRepository.getRecentLogs()
        assertThat(notificationLogs).hasSize(3)
        assertThat(notificationLogs.all { it.isGroup }).isTrue()
    }
    
    @Test
    fun testSilentNotificationFiltering() {
        // Given
        notificationService = startService(
            FamilyGuardNotificationService::class.java
        ) as FamilyGuardNotificationService
        
        // When - Post silent notification
        postSilentNotification("com.android.systemui", "System update available")
        
        // Then - Should still be captured
        val notificationLogs = NotificationRepository.getRecentLogs()
        assertThat(notificationLogs).hasSize(1)
        assertThat(notificationLogs[0].isSilent).isTrue()
    }
}
```

### 2. Notification Permission Tests
```kotlin
@Test
fun testNotificationPermissionGranted() {
    // Given
    val notificationManager = context.getSystemService(
        NotificationManager::class.java
    ) as NotificationManager
    
    // When - Check permission
    val hasPermission = notificationManager
        .isNotificationPolicyAccessGranted
    
    // Then
    if (!hasPermission) {
        // Request permission
        val intent = Intent(Settings.ACTION_NOTIFICATION_LISTENER_SETTINGS)
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        context.startActivity(intent)
        
        // Test should wait for manual permission grant
        fail("Notification permission not granted. Please grant manually.")
    }
    
    assertThat(hasPermission).isTrue()
}
```

## VPN Service Tests

### 1. DNS Filtering Tests
```kotlin
@RunWith(AndroidJUnit4::class)
class VpnServiceTest : BaseServiceTest() {
    
    private lateinit var vpnService: FamilyGuardVpnService
    
    @Test
    fun testDnsQueryInterception() {
        // Given
        vpnService = startService(
            FamilyGuardVpnService::class.java
        ) as FamilyGuardVpnService
        
        val testDomains = listOf(
            "facebook.com" to true,    // Should be blocked
            "google.com" to false,     // Should be allowed
            "pornhub.com" to true      // Should be blocked
        )
        
        // When & Then
        testDomains.forEach { (domain, shouldBlock) ->
            val dnsPacket = createDnsQueryPacket(domain)
            val intercepted = vpnService.interceptDnsQuery(dnsPacket)
            
            assertThat(intercepted).isEqualTo(shouldBlock)
            
            if (shouldBlock) {
                val blockedLog = VpnLogRepository.getRecentBlocks()
                assertThat(blockedLog).anyMatch { it.domain == domain }
            }
        }
    }
    
    @Test
    fun testUrlLogging() {
        // Given
        vpnService = startService(
            FamilyGuardVpnService::class.java
        ) as FamilyGuardVpnService
        
        val testUrls = listOf(
            "https://www.youtube.com/watch?v=test",
            "https://instagram.com/p/abcd1234",
            "https://twitter.com/user/status/123456"
        )
        
        // When - Simulate HTTP requests
        testUrls.forEach { url ->
            simulateHttpRequest(url)
        }
        
        // Then - Verify logging
        val urlLogs = BrowserHistoryRepository.getRecentUrls()
        assertThat(urlLogs).hasSize(testUrls.size)
        testUrls.forEach { url ->
            assertThat(urlLogs).anyMatch { it.url.contains(url) }
        }
    }
    
    @Test
    fun testVpnServiceStability() {
        // Given
        vpnService = startService(
            FamilyGuardVpnService::class.java
        ) as FamilyGuardVpnService
        
        // When - Simulate network changes
        simulateNetworkChange(NetworkType.WIFI)
        Thread.sleep(1000)
        
        simulateNetworkChange(NetworkType.MOBILE)
        Thread.sleep(1000)
        
        simulateNetworkChange(NetworkType.NONE)
        Thread.sleep(1000)
        
        simulateNetworkChange(NetworkType.WIFI)
        Thread.sleep(1000)
        
        // Then - Service should remain stable
        assertThat(vpnService.isRunning).isTrue()
        assertThat(vpnService.hasCrashes).isFalse()
    }
}
```

### 2. VPN Performance Tests
```kotlin
@Test
fun testVpnLatencyImpact() {
    // Given
    vpnService = startService(
        FamilyGuardVpnService::class.java
    ) as FamilyGuardVpnService
    
    // When - Measure latency without VPN
    val latencyWithoutVpn = measureNetworkLatency("google.com")
    
    // When - Measure latency with VPN
    vpnService.startVpn()
    val latencyWithVpn = measureNetworkLatency("google.com")
    
    // Then - VPN should not add significant latency
    val latencyIncrease = latencyWithVpn - latencyWithoutVpn
    assertThat(latencyIncrease).isLessThan(100) // Less than 100ms increase
}

@Test
fun testVpnDataUsage() {
    // Given
    vpnService = startService(
        FamilyGuardVpnService::class.java
    ) as FamilyGuardVpnService
    
    val startBytes = getNetworkUsageBytes()
    
    // When - Generate network traffic through VPN
    generateTestTraffic(10 * 1024 * 1024) // 10MB
    
    val endBytes = getNetworkUsageBytes()
    val dataUsed = endBytes - startBytes
    
    // Then - VPN should not significantly increase data usage
    val overhead = dataUsed - (10 * 1024 * 1024)
    val overheadPercentage = (overhead.toDouble() / (10 * 1024 * 1024)) * 100
    
    assertThat(overheadPercentage).isLessThan(5.0) // Less than 5% overhead
}
```

## Location Service Tests

### 1. Location Tracking Tests
```kotlin
@RunWith(AndroidJUnit4::class)
class LocationServiceTest : BaseServiceTest() {
    
    private lateinit var locationService: FamilyGuardLocationService
    
    @Test
    fun testLocationUpdates() {
        // Given
        locationService = startService(
            FamilyGuardLocationService::class.java
        ) as FamilyGuardLocationService
        
        val testLocations = listOf(
            TestLocation(lat = 37.7749, lon = -122.4194, accuracy = 10.0),
            TestLocation(lat = 37.7750, lon = -122.4195, accuracy = 15.0),
            TestLocation(lat = 37.7751, lon = -122.4196, accuracy = 8.0)
        )
        
        // When - Simulate location updates
        testLocations.forEach { testLocation ->
            simulateLocationUpdate(testLocation)
            Thread.sleep(1000) // Wait for processing
        }
        
        // Then - Verify locations stored
        val storedLocations = LocationRepository.getRecentPoints()
        assertThat(storedLocations).hasSize(testLocations.size)
        
        testLocations.forEachIndexed { index, testLocation ->
            assertThat(storedLocations[index].latitude)
                .isWithin(0.0001).of(testLocation.lat)
            assertThat(storedLocations[index].longitude)
                .isWithin(0.0001).of(testLocation.lon)
        }
    }
    
    @Test
    fun testGeofenceTriggering() {
        // Given
        locationService = startService(
            FamilyGuardLocationService::class.java
        ) as FamilyGuardLocationService
        
        val homeGeofence = Geofence(
            name = "Home",
            latitude = 37.7749,
            longitude = -122.4194,
            radius = 100.0f,
            type = GeofenceType.SAFE
        )
        
        locationService.addGeofence(homeGeofence)
        
        // When - Enter geofence
        simulateLocationUpdate(TestLocation(
            lat = 37.77491,
            lon = -122.41941,
            accuracy = 10.0
        ))
        
        // Then - Should trigger entry
        val alerts = AlertRepository.getRecentAlerts()
        assertThat(alerts).hasSize(1)
        assertThat(alerts[0].alertType).isEqualTo("geofence_entry")
        assertThat(alerts[0].data).contains("Home")
        
        // When - Exit geofence
        simulateLocationUpdate(TestLocation(
            lat = 37.7800,
            lon = -122.4300,
            accuracy = 10.0
        ))
        
        // Then - Should trigger exit
        val updatedAlerts = AlertRepository.getRecentAlerts()
        assertThat(updatedAlerts).hasSize(2)
        assertThat(updatedAlerts[1].alertType).isEqualTo("geofence_exit")
    }
    
    @Test
    fun testBackgroundLocationPermission() {
        // Given
        locationService = startService(
            FamilyGuardLocationService::class.java
        ) as FamilyGuardLocationService
        
        // When - Check permission
        val hasBackgroundPermission = locationService
            .hasBackgroundLocationPermission()
        
        // Then
        if (!hasBackgroundPermission) {
            // Request permission
            val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS)
            intent.data = Uri.parse("package:${context.packageName}")
            intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
            context.startActivity(intent)
            
            fail("Background location permission not granted. Please grant 'Allow all the time'.")
        }
        
        assertThat(hasBackgroundPermission).isTrue()
    }
}
```

### 2. Location Accuracy Tests
```kotlin
@Test
fun testLocationAccuracyFiltering() {
    // Given
    locationService = startService(
        FamilyGuardLocationService::class.java
    ) as FamilyGuardLocationService
    
    val locations = listOf(
        LocationPoint(accuracy = 5.0f),   // Good
        LocationPoint(accuracy = 50.0f),  // Medium
        LocationPoint(accuracy = 200.0f), // Poor
        LocationPoint(accuracy = 10.0f),  // Good
        LocationPoint(accuracy = 100.0f)  // Poor
    )
    
    // When - Filter with threshold
    val filtered = locationService.filterByAccuracy(locations, 30.0f)
    
    // Then
    assertThat(filtered).hasSize(2) // Only accuracy < 30
    filtered.forEach { location ->
        assertThat(location.accuracy).isLessThan(30.0f)
    }
}

@Test
fun testBatteryOptimizedLocationUpdates() {
    // Given
    locationService = startService(
        FamilyGuardLocationService::class.java
    ) as FamilyGuardLocationService
    
    // When - Start battery optimized tracking
    locationService.startBatteryOptimizedTracking()
    
    // Then - Should use appropriate interval
    val updateInterval = locationService.getUpdateInterval()
    
    // When device is stationary
    simulateDeviceStationary()
    assertThat(updateInterval).isGreaterThan(300000L) // > 5 minutes
    
    // When device is moving
    simulateDeviceMoving()
    val movingInterval = locationService.getUpdateInterval()
    assertThat(movingInterval).isLessThan(60000L) // < 1 minute
}
```

## Media Capture Service Tests

### 1. Screen Capture Tests
```kotlin
@RunWith(AndroidJUnit4::class)
class MediaCaptureServiceTest : BaseServiceTest() {
    
    private lateinit var mediaService: FamilyGuardMediaService
    
    @Test
    fun testScreenshotCapture() {
        // Given
        mediaService = startService(
            FamilyGuardMediaService::class.java
        ) as FamilyGuardMediaService
        
        // When - Capture screenshot
        val screenshotResult = mediaService.captureScreenshot()
        
        // Then
        assertThat(screenshotResult.success).isTrue()
        assertThat(screenshotResult.filePath).isNotEmpty()
        
        val screenshotFile = File(screenshotResult.filePath)
        assertThat(screenshotFile.exists()).isTrue()
        assertThat(screenshotFile.length()).isGreaterThan(0L)
        
        // Verify database record
        val mediaCaptures = MediaRepository.getRecentCaptures()
        assertThat(mediaCaptures).hasSize(1)
        assertThat(mediaCaptures[0].mediaType).isEqualTo("screenshot")
    }
    
    @Test
    fun testScreenRecording() {
        // Given
        mediaService = startService(
            FamilyGuardMediaService::class.java
        ) as FamilyGuardMediaService
        
        // When - Start recording
        val startResult = mediaService.startScreenRecording(duration = 5000L)
        assertThat(startResult.success).isTrue()
        
        // Wait for recording to complete
        Thread.sleep(6000L)
        
        // Then
        val mediaCaptures = MediaRepository.getRecentCaptures()
        assertThat(mediaCaptures).hasSize(1)
        assertThat(mediaCaptures[0].mediaType).isEqualTo("screen_recording")
        assertThat(mediaCaptures[0].duration).isGreaterThan(0L)
    }
    
    @Test
    fun testCameraPhotoCapture() {
        // Given
        mediaService = startService(
            FamilyGuardMediaService::class.java
        ) as FamilyGuardMediaService
        
        // When - Capture photo
        val photoResult = mediaService.capturePhoto(camera = "front")
        
        // Then
        assertThat(photoResult.success).isTrue()
        assertThat(photoResult.filePath).isNotEmpty()
        
        val photoFile = File(photoResult.filePath)
        assertThat(photoFile.exists()).isTrue()
        
        val mediaCaptures = MediaRepository.getRecentCaptures()
        assertThat(mediaCaptures).hasSize(1)
        assertThat(mediaCaptures[0].mediaType).isEqualTo("camera_photo")
    }
}
```

### 2. Audio Recording Tests
```kotlin
@Test
fun testMicrophoneRecording() {
    // Given
    mediaService = startService(
        FamilyGuardMediaService::class.java
    ) as FamilyGuardMediaService
    
    // When - Start recording
    val startResult = mediaService.startAudioRecording(duration = 3000L)
    assertThat(startResult.success).isTrue()
    
    // Generate some test audio
    generateTestAudio(2000L)
    
    // Wait for recording to complete
    Thread.sleep(4000L)
    
    // Then
    val mediaCaptures = MediaRepository.getRecentCaptures()
    assertThat(mediaCaptures).hasSize(1)
    assertThat(mediaCaptures[0].mediaType).isEqualTo("mic_audio")
    assertThat(mediaCaptures[0].duration).isGreaterThan(0L)
}

@Test
fun testLiveAudioStreaming() {
    // Given
    mediaService = startService(
        FamilyGuardMediaService::class.java
    ) as FamilyGuardMediaService
    
    val webSocketClient = TestWebSocketClient()
    
    // When - Start live listen
    val startResult = mediaService.startLiveListen(
        duration = 5000L,
        webSocketClient = webSocketClient
    )
    assertThat(startResult.success).isTrue()
    
    // Then - Should receive audio chunks
    eventually(10, TimeUnit.SECONDS) {
        val audioChunks = webSocketClient.getReceivedChunks()
        assertThat(audioChunks).isNotEmpty()
        assertThat(audioChunks.size).isGreaterThan(1)
    }
}
```

## Device Admin Tests

### 1. Device Owner Functionality Tests
```kotlin
@RunWith(AndroidJUnit4::class)
class DeviceAdminTest : BaseServiceTest() {
    
    private lateinit var deviceAdmin: FamilyGuardDeviceAdminReceiver
    
    @Test
    fun testDeviceOwnerProvisioning() {
        // Given
        deviceAdmin = FamilyGuardDeviceAdminReceiver()
        
        // When - Check if device owner
        val isDeviceOwner = deviceAdmin.isDeviceOwner(context)
        
        // Then
        if (!isDeviceOwner) {
            // Provide provisioning instructions
            println("Device Owner not set. Use ADB command:")
            println("adb shell dpm set-device-owner " +
                "com.familyguard.child/.service.admin.FamilyGuardDeviceAdminReceiver")
            fail("Device Owner provisioning required for full test")
        }
        
        assertThat(isDeviceOwner).isTrue()
    }
    
    @Test
    fun testRemoteLock() {
        // Given
        deviceAdmin = FamilyGuardDeviceAdminReceiver()
        
        // When - Lock device
        val lockResult = deviceAdmin.lockDevice(context, "Test lock")
        
        // Then
        assertThat(lockResult.success).isTrue()
        
        // Device should be locked
        val keyguardManager = context.getSystemService(
            Context.KEYGUARD_SERVICE
        ) as KeyguardManager
        
        assertThat(keyguardManager.isDeviceLocked).isTrue()
        
        // Cleanup - unlock for further tests
        deviceAdmin.unlockDevice(context, "test_password")
    }
    
    @Test
    fun testAppUninstallPrevention() {
        // Given
        deviceAdmin = FamilyGuardDeviceAdminReceiver()
        
        // When - Try to uninstall (should fail)
        val uninstallIntent = Intent(Intent.ACTION_DELETE)
        uninstallIntent.data = Uri.parse("package:${context.packageName}")
        
        val result = try {
            context.startActivity(uninstallIntent)
            false
        } catch (e: SecurityException) {
            true // Expected - uninstall prevented
        }
        
        // Then
        assertThat(result).isTrue()
    }
}
```

### 2. Policy Enforcement Tests
```kotlin
@Test
fun testAppBlocking() {
    // Given
    deviceAdmin = FamilyGuardDeviceAdminReceiver()
    
    // When - Block TikTok
    val blockResult = deviceAdmin.blockApplication(
        context, 
        "com.zhiliaoapp.musically" // TikTok
    )
    
    assertThat(blockResult.success).isTrue()
    
    // Then - App should be blocked
    val packageManager = context.packageManager
    val appInfo = try {
        packageManager.getApplicationInfo(
            "com.zhiliaoapp.musically",
            0
        )
        false // App still accessible
    } catch (e: PackageManager.NameNotFoundException) {
        true // App blocked/not found
    }
    
    assertThat(appInfo).isTrue()
}

@Test
fun testCameraDisable() {
    // Given
    deviceAdmin = FamilyGuardDeviceAdminReceiver()
    
    // When - Disable camera
    val disableResult = deviceAdmin.disableCamera(context, true)
    assertThat(disableResult.success).isTrue()
    
    // Then - Camera should be disabled
    val cameraManager = context.getSystemService(
        Context.CAMERA_SERVICE
    ) as CameraManager
    
    try {
        cameraManager.cameraIdList // This should fail
        fail("Camera should be disabled")
    } catch (e: CameraAccessException) {
        // Expected - camera disabled
        assertThat(e.reason).isEqualTo(
            CameraAccessException.CAMERA_DISABLED
        )
    }
    
    // Cleanup - re-enable camera
    deviceAdmin.disableCamera(context, false)
}
```

## Test Utilities

### 1. Mock Service Helpers
```kotlin
object ServiceTestUtils {
    
    fun createMockAccessibilityNode(
        packageName: String,
        text: String = "",
        className: String = ""
    ): AccessibilityNodeInfo {
        val node = AccessibilityNodeInfo.obtain()
        
        // Set node properties
        node.packageName = packageName
        node.className = className
        node.text = text
        
        // Add to window
        node.setSource(View(InstrumentationRegistry.getInstrumentation().context))
        
        return node
    }
    
    fun createTestNotification(
        packageName: String,
        title: String,
        text: String,
        timestamp: Long = System.currentTimeMillis()
    ): Notification {
        return Notification.Builder(context, "test_channel")
            .setContentTitle(title)
            .setContentText(text)
            .setSmallIcon(android.R.drawable.ic_dialog_info)
            .setWhen(timestamp)
            .build()
    }
    
    fun simulateLocationUpdate(location: TestLocation) {
        val locationManager = context.getSystemService(
            Context.LOCATION_SERVICE
        ) as LocationManager
        
        val mockLocation = Location(LocationManager.GPS_PROVIDER).apply {
            latitude = location.lat
            longitude = location.lon
            accuracy = location.accuracy
            time = System.currentTimeMillis()
        }
        
        // This requires mock location provider setup
        locationManager.setTestProviderLocation(
            LocationManager.GPS_PROVIDER,
            mockLocation
        )
    }
}
```

### 2. Test Data Management
```kotlin
object TestDataManager {
    
    private val testDatabase = mutableMapOf<String, Any>()
    
    fun initializeTestDatabase() {
        // Create in-memory Room database for testing
        val database = Room.inMemoryDatabaseBuilder(
            InstrumentationRegistry.getInstrumentation().targetContext,
            FamilyGuardDatabase::class.java
        ).build()
        
        testDatabase["room_db"] = database
    }
    
    fun clearTestData() {
        // Clear all test data
        (testDatabase["room_db"] as? FamilyGuardDatabase)?.clearAllTables()
        testDatabase.clear()
    }
    
    fun insertTestSocialCapture(capture: SocialCapture) {
        val dao = (testDatabase["room_db"] as FamilyGuardDatabase)
            .socialCaptureDao()
        runBlocking {
            dao.insert(capture)
        }
    }
    
    fun getTestSocialCaptures(): List<SocialCapture> {
        val dao = (testDatabase["room_db"] as FamilyGuardDatabase)
            .socialCaptureDao()
        return runBlocking {
            dao.getRecentCaptures(100)
        }
    }
}
```

## Test Execution Commands

### 1. Run Specific Test Classes
```bash
# Run AccessibilityService tests
./gradlew connectedDebugAndroidTest \
  --tests "*AccessibilityServiceTest"

# Run NotificationService tests  
./gradlew connectedDebugAndroidTest \
  --tests "*NotificationServiceTest"

# Run VpnService tests
./gradlew connectedDebugAndroidTest \
  --tests "*VpnServiceTest"

# Run all service tests
./gradlew connectedDebugAndroidTest \
  --tests "*ServiceTest"
```

### 2. Test with Specific Devices
```bash
# Test on specific API level
./gradlew connectedDebugAndroidTest \
  -Pandroid.testInstrumentationRunnerArguments.androidx.benchmark.enabled=true \
  -Pandroid.testInstrumentationRunnerArguments.androidx.benchmark.output.enable=true

# Test with coverage
./gradlew connectedDebugAndroidTest \
  -Pandroid.testInstrumentationRunnerArguments.coverage=true

# Test on multiple devices
./gradlew connectedDebugAndroidTest \
  -PtestDevice=Pixel_6_API_33 \
  -PtestDevice=Pixel_3_API_30 \
  -PtestDevice=Galaxy_S10_API_29
```

### 3. Generate Test Reports
```bash
# Generate HTML test report
./gradlew connectedDebugAndroidTest \
  -Pandroid.testInstrumentationRunnerArguments.reportDir=build/reports/androidTests

# Generate JUnit XML reports
./gradlew connectedDebugAndroidTest \
  -Pandroid.testInstrumentationRunnerArguments.junitOutputDir=build/test-results

# Generate Allure reports
./gradlew connectedDebugAndroidTest allureReport
```

## Test Maintenance

### 1. Service Test Checklist
- [ ] Service starts and stops correctly
- [ ] Permissions are properly handled
- [ ] Data persistence works across restarts
- [ ] Error handling is robust
- [ ] Battery impact is minimal
- [ ] Memory usage is optimized
- [ ] Network usage is efficient

### 2. Test Data Cleanup
```kotlin
@After
fun cleanup() {
    // Stop all services
    ServiceController.stopAllServices()
    
    // Clear test databases
    TestDatabase.clearAll()
    
    // Remove test files
    TestFileUtils.cleanupTestFiles()
    
    // Reset system settings
    TestSystemUtils.resetSettings()
    
    // Clear caches
    context.cacheDir.deleteRecursively()
}
```

### 3. Test Documentation
- Document required permissions for each test
- Note device requirements (API level, features)
- Include setup instructions for manual tests
- Document known limitations or issues
- Provide examples of expected output

## Conclusion

These instrumentation tests ensure FamilyGuard services work correctly in real Android environments. They validate system integration, permission handling, and service reliability across different Android versions and device configurations.

Regular execution of these tests, especially before releases, ensures monitoring services maintain their functionality and reliability while minimizing impact on device performance and battery life.

---

**Test Environment Requirements**:
- Android 8.0+ (API 26+)
- USB Debugging enabled
- Test device with required sensors
- Internet connection for network tests
- Sufficient storage for media tests

**Note**: Some tests require manual intervention for permission granting or Device Owner setup. These should be documented and automated where possible.