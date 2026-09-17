# FamilyGuard UI Tests with Jetpack Compose

## Overview
This document outlines the UI testing strategy for FamilyGuard Parent app using Jetpack Compose UI Test framework. Tests cover navigation, user interactions, data display, and accessibility.

## Test Architecture

### 1. Test Environment Setup
```kotlin
@HiltAndroidTest
@UninstallModules(AppModule::class)
class ComposeTestBase {
    
    @get:Rule
    val hiltRule = HiltAndroidRule(this)
    
    @get:Rule
    val composeTestRule = createComposeRule()
    
    @Inject
    lateinit var testRepository: TestRepository
    
    @Before
    fun setup() {
        hiltRule.inject()
        
        // Set up test data
        testRepository.initializeTestData()
        
        // Set up mock responses
        MockWebServer().apply {
            start()
            enqueueMockResponses()
        }
        
        // Set test mode
        TestMode.enable()
    }
    
    @After
    fun cleanup() {
        // Clear test data
        testRepository.clearAll()
        
        // Reset test mode
        TestMode.reset()
    }
}
```

### 2. Test Data Builders
```kotlin
object TestDataBuilders {
    
    fun buildTestDevice(): Device {
        return Device(
            id = "test_device_123",
            deviceId = "android_device_123",
            childName = "Test Child",
            status = DeviceStatus.ONLINE,
            batteryLevel = 85,
            lastSeen = System.currentTimeMillis(),
            location = TestLocation(
                latitude = 37.7749,
                longitude = -122.4194
            )
        )
    }
    
    fun buildTestSocialCaptures(count: Int = 5): List<SocialCapture> {
        return (1..count).map { index ->
            SocialCapture(
                id = "social_$index",
                appPackage = "com.whatsapp",
                appName = "WhatsApp",
                contentType = "text",
                sender = "+1234567890",
                receiver = "+0987654321",
                messageText = "Test message $index",
                timestamp = System.currentTimeMillis() - (index * 3600000L),
                isEncrypted = false,
                isSent = index % 2 == 0
            )
        }
    }
    
    fun buildTestLocationHistory(count: Int = 10): List<LocationPoint> {
        return (1..count).map { index ->
            LocationPoint(
                id = "loc_$index",
                latitude = 37.7749 + (index * 0.001),
                longitude = -122.4194 + (index * 0.001),
                accuracy = 10.0f,
                timestamp = System.currentTimeMillis() - (index * 300000L),
                provider = "gps",
                isMoving = index % 3 == 0
            )
        }
    }
}
```

## Dashboard UI Tests

### 1. Main Dashboard Tests
```kotlin
@HiltAndroidTest
class DashboardScreenTest : ComposeTestBase() {
    
    @Test
    fun testDashboardLoadsDevices() {
        // Given - Test devices
        val testDevices = listOf(
            buildTestDevice().copy(childName = "Alice"),
            buildTestDevice().copy(childName = "Bob"),
            buildTestDevice().copy(childName = "Charlie")
        )
        
        testRepository.setDevices(testDevices)
        
        // When - Load dashboard
        composeTestRule.setContent {
            FamilyGuardTheme {
                DashboardScreen(navController = rememberNavController())
            }
        }
        
        // Then - Verify devices displayed
        testDevices.forEach { device ->
            composeTestRule
                .onNodeWithText(device.childName)
                .assertIsDisplayed()
            
            composeTestRule
                .onNodeWithText("${device.batteryLevel}%")
                .assertIsDisplayed()
        }
    }
    
    @Test
    fun testDeviceStatusIndicators() {
        // Given - Devices with different statuses
        val devices = listOf(
            buildTestDevice().copy(status = DeviceStatus.ONLINE),
            buildTestDevice().copy(status = DeviceStatus.OFFLINE),
            buildTestDevice().copy(status = DeviceStatus.ALERT)
        )
        
        testRepository.setDevices(devices)
        
        // When - Load dashboard
        composeTestRule.setContent {
            FamilyGuardTheme {
                DashboardScreen(navController = rememberNavController())
            }
        }
        
        // Then - Verify status indicators
        composeTestRule
            .onNodeWithTag("status_online")
            .assertIsDisplayed()
            
        composeTestRule
            .onNodeWithTag("status_offline")  
            .assertIsDisplayed()
            
        composeTestRule
            .onNodeWithTag("status_alert")
            .assertIsDisplayed()
    }
    
    @Test
    fun testDashboardNavigation() {
        // Given
        composeTestRule.setContent {
            FamilyGuardTheme {
                DashboardScreen(navController = rememberNavController())
            }
        }
        
        // When - Click on device card
        composeTestRule
            .onNodeWithText("Test Child")
            .performClick()
        
        // Then - Should navigate to device detail
        composeTestRule
            .onNodeWithText("Device Details")
            .assertIsDisplayed()
    }
    
    @Test
    fun testQuickActions() {
        // Given
        composeTestRule.setContent {
            FamilyGuardTheme {
                DashboardScreen(navController = rememberNavController())
            }
        }
        
        // When - Click quick actions
        composeTestRule
            .onNodeWithText("Lock Device")
            .performClick()
            
        // Then - Should show confirmation
        composeTestRule
            .onNodeWithText("Confirm Lock")
            .assertIsDisplayed()
            
        // When - Confirm
        composeTestRule
            .onNodeWithText("Confirm")
            .performClick()
            
        // Then - Should show success
        composeTestRule
            .onNodeWithText("Device Locked")
            .assertIsDisplayed()
    }
}
```

### 2. Real-time Updates Tests
```kotlin
@Test
fun testLiveLocationUpdates() {
    // Given
    val device = buildTestDevice()
    testRepository.setDevices(listOf(device))
    
    composeTestRule.setContent {
        FamilyGuardTheme {
            DashboardScreen(navController = rememberNavController())
        }
    }
    
    // When - Simulate location update
    val newLocation = TestLocation(
        latitude = 37.7750,
        longitude = -122.4195
    )
    
    testRepository.updateDeviceLocation(
        deviceId = device.id,
        location = newLocation
    )
    
    // Then - UI should update
    composeTestRule.waitUntil(5000) {
        composeTestRule
            .onNodeWithText("Location updated")
            .assertExists()
    }
}

@Test
fun testBatteryUpdates() {
    // Given
    val device = buildTestDevice().copy(batteryLevel = 100)
    testRepository.setDevices(listOf(device))
    
    composeTestRule.setContent {
        FamilyGuardTheme {
            DashboardScreen(navController = rememberNavController())
        }
    }
    
    // When - Simulate battery drain
    testRepository.updateDeviceBattery(device.id, 85)
    
    // Then - Battery indicator should update
    composeTestRule.waitUntil(3000) {
        composeTestRule
            .onNodeWithText("85%")
            .assertIsDisplayed()
    }
}
```

## Device Detail Screen Tests

### 1. Device Monitoring Views
```kotlin
@HiltAndroidTest
class DeviceDetailScreenTest : ComposeTestBase() {
    
    @Test
    fun testDeviceOverviewTab() {
        // Given
        val device = buildTestDevice()
        testRepository.setDevices(listOf(device))
        
        // When - Navigate to device detail
        composeTestRule.setContent {
            FamilyGuardTheme {
                NavHost(
                    navController = rememberNavController(),
                    startDestination = "device/${device.id}"
                ) {
                    composable("device/{deviceId}") {
                        DeviceDetailScreen()
                    }
                }
            }
        }
        
        // Then - Verify device info
        composeTestRule
            .onNodeWithText(device.childName)
            .assertIsDisplayed()
            
        composeTestRule
            .onNodeWithText("Battery: ${device.batteryLevel}%")
            .assertIsDisplayed()
            
        composeTestRule
            .onNodeWithText("Status: ${device.status}")
            .assertIsDisplayed()
    }
    
    @Test
    fun testTabNavigation() {
        // Given
        val device = buildTestDevice()
        testRepository.setDevices(listOf(device))
        
        composeTestRule.setContent {
            FamilyGuardTheme {
                DeviceDetailScreen(
                    deviceId = device.id,
                    navController = rememberNavController()
                )
            }
        }
        
        // When - Click Social tab
        composeTestRule
            .onNodeWithText("Social")
            .performClick()
        
        // Then - Should show social content
        composeTestRule
            .onNodeWithText("Social Media Activity")
            .assertIsDisplayed()
        
        // When - Click Location tab
        composeTestRule
            .onNodeWithText("Location")
            .performClick()
        
        // Then - Should show map
        composeTestRule
            .onNodeWithTag("location_map")
            .assertIsDisplayed()
        
        // When - Click Apps tab
        composeTestRule
            .onNodeWithText("Apps")
            .performClick()
        
        // Then - Should show app usage
        composeTestRule
            .onNodeWithText("App Usage")
            .assertIsDisplayed()
    }
    
    @Test
    fun testRemoteCommands() {
        // Given
        val device = buildTestDevice()
        testRepository.setDevices(listOf(device))
        
        composeTestRule.setContent {
            FamilyGuardTheme {
                DeviceDetailScreen(
                    deviceId = device.id,
                    navController = rememberNavController()
                )
            }
        }
        
        // Test Lock command
        composeTestRule
            .onNodeWithText("Lock Device")
            .performClick()
            
        composeTestRule
            .onNodeWithText("Enter lock message")
            .performTextInput("Parent lock")
            
        composeTestRule
            .onNodeWithText("Send Lock")
            .performClick()
            
        composeTestRule
            .onNodeWithText("Lock command sent")
            .assertIsDisplayed()
        
        // Test Ring command
        composeTestRule
            .onNodeWithText("Ring Device")
            .performClick()
            
        composeTestRule
            .onNodeWithText("Ring for 60 seconds")
            .assertIsDisplayed()
    }
}
```

### 2. Social Media Viewer Tests
```kotlin
@Test
fun testSocialMediaMessageList() {
    // Given
    val socialCaptures = buildTestSocialCaptures(10)
    testRepository.setSocialCaptures(socialCaptures)
    
    composeTestRule.setContent {
        FamilyGuardTheme {
            SocialMediaScreen(
                deviceId = "test_device_123",
                navController = rememberNavController()
            )
        }
    }
    
    // Then - Verify messages displayed
    socialCaptures.forEach { capture ->
        composeTestRule
            .onNodeWithText(capture.messageText!!, substring = true)
            .assertIsDisplayed()
    }
}

@Test
fun testSocialMediaFiltering() {
    // Given
    val captures = listOf(
        buildTestSocialCapture(appPackage = "com.whatsapp"),
        buildTestSocialCapture(appPackage = "com.instagram.android"),
        buildTestSocialCapture(appPackage = "com.tiktok"),
        buildTestSocialCapture(appPackage = "com.facebook.katana")
    )
    
    testRepository.setSocialCaptures(captures)
    
    composeTestRule.setContent {
        FamilyGuardTheme {
            SocialMediaScreen(
                deviceId = "test_device_123",
                navController = rememberNavController()
            )
        }
    }
    
    // When - Filter by WhatsApp
    composeTestRule
        .onNodeWithText("Filter")
        .performClick()
        
    composeTestRule
        .onNodeWithText("WhatsApp")
        .performClick()
    
    // Then - Only WhatsApp messages shown
    composeTestRule
        .onNodeWithText("WhatsApp messages")
        .assertIsDisplayed()
        
    composeTestRule
        .onNodeWithText("Instagram")
        .assertDoesNotExist()
}

@Test
fun testMessageSearch() {
    // Given
    val captures = buildTestSocialCaptures(20)
    testRepository.setSocialCaptures(captures)
    
    composeTestRule.setContent {
        FamilyGuardTheme {
            SocialMediaScreen(
                deviceId = "test_device_123",
                navController = rememberNavController()
            )
        }
    }
    
    // When - Search for specific text
    composeTestRule
        .onNodeWithTag("search_field")
        .performTextInput("message 5")
    
    // Then - Only matching messages shown
    composeTestRule
        .onNodeWithText("message 5")
        .assertIsDisplayed()
        
    composeTestRule
        .onNodeWithText("message 1")
        .assertDoesNotExist()
}
```

### 3. Location Map Tests
```kotlin
@Test
fun testLocationMapDisplay() {
    // Given
    val locationHistory = buildTestLocationHistory(15)
    testRepository.setLocationHistory(locationHistory)
    
    composeTestRule.setContent {
        FamilyGuardTheme {
            LocationScreen(
                deviceId = "test_device_123",
                navController = rememberNavController()
            )
        }
    }
    
    // Then - Map should be displayed
    composeTestRule
        .onNodeWithTag("location_map")
        .assertIsDisplayed()
    
    // Verify location markers
    composeTestRule
        .onAllNodesWithTag("location_marker")
        .assertCountEquals(15)
}

@Test
fun testGeofenceManagement() {
    // Given
    composeTestRule.setContent {
        FamilyGuardTheme {
            LocationScreen(
                deviceId = "test_device_123",
                navController = rememberNavController()
            )
        }
    }
    
    // When - Add geofence
    composeTestRule
        .onNodeWithText("Add Geofence")
        .performClick()
    
    composeTestRule
        .onNodeWithText("Geofence Name")
        .performTextInput("Home")
    
    composeTestRule
        .onNodeWithText("Radius (meters)")
        .performTextInput("100")
    
    composeTestRule
        .onNodeWithText("Save Geofence")
        .performClick()
    
    // Then - Geofence should appear
    composeTestRule
        .onNodeWithText("Home Geofence")
        .assertIsDisplayed()
        
    composeTestRule
        .onNodeWithTag("geofence_circle")
        .assertIsDisplayed()
    
    // When - Delete geofence
    composeTestRule
        .onNodeWithText("Home")
        .performTouchInput {
            longClick()
        }
    
    composeTestRule
        .onNodeWithText("Delete")
        .performClick()
    
    // Then - Geofence removed
    composeTestRule
        .onNodeWithText("Home")
        .assertDoesNotExist()
}
```

### 4. App Usage Screen Tests
```kotlin
@Test
fun testAppUsageStatistics() {
    // Given
    val appUsage = listOf(
        AppUsage(appPackage = "com.tiktok", usageTime = 120, date = "2024-01-01"),
        AppUsage(appPackage = "com.youtube", usageTime = 90, date = "2024-01-01"),
        AppUsage(appPackage = "com.instagram", usageTime = 60, date = "2024-01-01"),
        AppUsage(appPackage = "com.whatsapp", usageTime = 45, date = "2024-01-01")
    )
    
    testRepository.setAppUsage(appUsage)
    
    composeTestRule.setContent {
        FamilyGuardTheme {
            AppUsageScreen(
                deviceId = "test_device_123",
                navController = rememberNavController()
            )
        }
    }
    
    // Then - Verify app usage displayed
    composeTestRule
        .onNodeWithText("TikTok: 2h 0m")
        .assertIsDisplayed()
        
    composeTestRule
        .onNodeWithText("YouTube: 1h 30m")
        .assertIsDisplayed()
    
    // Verify chart displayed
    composeTestRule
        .onNodeWithTag("usage_chart")
        .assertIsDisplayed()
}

@Test
fun testAppBlocking() {
    // Given
    composeTestRule.setContent {
        FamilyGuardTheme {
            AppUsageScreen(
                deviceId = "test_device_123",
                navController = rememberNavController()
            )
        }
    }
    
    // When - Block TikTok
    composeTestRule
        .onNodeWithText("TikTok")
        .performTouchInput {
            longClick()
        }
    
    composeTestRule
        .onNodeWithText("Block App")
        .performClick()
    
    composeTestRule
        .onNodeWithText("Block Duration")
        .performTextInput("2")
    
    composeTestRule
        .onNodeWithText("Block")
        .performClick()
    
    // Then - Should show blocked status
    composeTestRule
        .onNodeWithText("Blocked for 2 hours")
        .assertIsDisplayed()
        
    composeTestRule
        .onNodeWithTag("blocked_indicator")
        .assertIsDisplayed()
}
```

## Settings Screen Tests

### 1. Privacy Settings Tests
```kotlin
@HiltAndroidTest
class SettingsScreenTest : ComposeTestBase() {
    
    @Test
    fun testPrivacySettings() {
        // Given
        composeTestRule.setContent {
            FamilyGuardTheme {
                SettingsScreen(navController = rememberNavController())
            }
        }
        
        // When - Navigate to privacy
        composeTestRule
            .onNodeWithText("Privacy")
            .performClick()
        
        // Then - Verify privacy options
        composeTestRule
            .onNodeWithText("Data Retention")
            .assertIsDisplayed()
            
        composeTestRule
            .onNodeWithText("Encryption Settings")
            .assertIsDisplayed()
            
        composeTestRule
            .onNodeWithText("Consent Management")
            .assertIsDisplayed()
    }
    
    @Test
    fun testDataRetentionSettings() {
        // Given
        composeTestRule.setContent {
            FamilyGuardTheme {
                SettingsScreen(navController = rememberNavController())
            }
        }
        
        composeTestRule
            .onNodeWithText("Privacy")
            .performClick()
            
        composeTestRule
            .onNodeWithText("Data Retention")
            .performClick()
        
        // When - Change retention period
        composeTestRule
            .onNodeWithText("30 days")
            .performClick()
            
        composeTestRule
            .onNodeWithText("90 days")
            .performClick()
        
        // Then - Should update
        composeTestRule
            .onNodeWithText("Retention: 90 days")
            .assertIsDisplayed()
    }
    
    @Test
    fun testConsentManagement() {
        // Given
        composeTestRule.setContent {
            FamilyGuardTheme {
                SettingsScreen(navController = rememberNavController())
            }
        }
        
        composeTestRule
            .onNodeWithText("Privacy")
            .performClick()
            
        composeTestRule
            .onNodeWithText("Consent Management")
            .performClick()
        
        // Then - Verify consent records
        composeTestRule
            .onNodeWithText("Consent Records")
            .assertIsDisplayed()
            
        composeTestRule
            .onNodeWithText("View Consent")
            .assertIsDisplayed()
            
        composeTestRule
            .onNodeWithText("Export Consent")
            .assertIsDisplayed()
        
        // When - Export consent
        composeTestRule
            .onNodeWithText("Export Consent")
            .performClick()
        
        // Then - Should show export options
        composeTestRule
            .onNodeWithText("Export Format")
            .assertIsDisplayed()
            
        composeTestRule
            .onNodeWithText("PDF")
            .assertIsDisplayed()
            
        composeTestRule
            .onNodeWithText("JSON")
            .assertIsDisplayed()
    }
}
```

### 2. Notification Settings Tests
```kotlin
@Test
fun testNotificationSettings() {
    // Given
    composeTestRule.setContent {
        FamilyGuardTheme {
            SettingsScreen(navController = rememberNavController())
        }
    }
    
    composeTestRule
        .onNodeWithText("Notifications")
        .performClick()
    
    // Test alert toggles
    val alertTypes = listOf(
        "Geofence Alerts",
        "App Usage Alerts", 
        "Content Alerts",
        "Device Status Alerts"
    )
    
    alertTypes.forEach { alertType ->
        // Toggle off
        composeTestRule
            .onNodeWithText(alertType)
            .performClick()
            
        // Verify toggle state
        composeTestRule
            .onNodeWithTag("${alertType}_toggle")
            .assertIsOff()
            
        // Toggle back on
        composeTestRule
            .onNodeWithText(alertType)
            .performClick()
            
        composeTestRule
            .onNodeWithTag("${alertType}_toggle")
            .assertIsOn()
    }
}

@Test
fun testAlertFrequencySettings() {
    // Given
    composeTestRule.setContent {
        FamilyGuardTheme {
            SettingsScreen(navController = rememberNavController())
        }
    }
    
    composeTestRule
        .onNodeWithText("Notifications")
        .performClick()
        
    composeTestRule
        .onNodeWithText("Alert Frequency")
        .performClick()
    
    // When - Change frequency
    composeTestRule
        .onNodeWithText("Immediate")
        .performClick()
        
    composeTestRule
        .onNodeWithText("Hourly Digest")
        .performClick()
    
    // Then - Should update
    composeTestRule
        .onNodeWithText("Frequency: Hourly Digest")
        .assertIsDisplayed()
}
```

## Navigation Flow Tests

### 1. Complete User Journey Tests
```kotlin
@Test
fun testCompleteParentJourney() {
    // Given - Fresh app start
    composeTestRule.setContent {
        FamilyGuardTheme {
            FamilyGuardApp()
        }
    }
    
    // 1. Login
    composeTestRule
        .onNodeWithTag("email_field")
        .performTextInput("parent@example.com")
        
    composeTestRule
        .onNodeWithTag("password_field")
        .performTextInput("password123")
        
    composeTestRule
        .onNodeWithText("Login")
        .performClick()
    
    // Verify dashboard loads
    composeTestRule.waitUntil(5000) {
        composeTestRule
            .onNodeWithText("FamilyGuard Dashboard")
            .assertExists()
    }
    
    // 2. View device details
    composeTestRule
        .onNodeWithText("Test Child")
        .performClick()
        
    composeTestRule
        .onNodeWithText("Device Details")
        .assertIsDisplayed()
    
    // 3. Check social media
    composeTestRule
        .onNodeWithText("Social")
        .performClick()
        
    composeTestRule
        .onNodeWithText("Social Media Activity")
        .assertIsDisplayed()
    
    // 4. Check location
    composeTestRule
        .onNodeWithText("Location")
        .performClick()
        
    composeTestRule
        .onNodeWithTag("location_map")
        .assertIsDisplayed()
    
    // 5. Send remote command
    composeTestRule
        .onNodeWithText("Remote Commands")
        .performClick()
        
    composeTestRule
        .onNodeWithText("Lock Device")
        .performClick()
        
    composeTestRule
        .onNodeWithText("Lock command sent")
        .assertIsDisplayed()
    
    // 6. Return to dashboard
    composeTestRule.pressBack()
    composeTestRule.pressBack()
    
    composeTestRule
        .onNodeWithText("FamilyGuard Dashboard")
        .assertIsDisplayed()
}

@Test
fun testBiometricAuthenticationFlow() {
    // Given - App with biometric auth enabled
    composeTestRule.setContent {
        FamilyGuardTheme {
            FamilyGuardApp()
        }
    }
    
    // When - Try to access sensitive area
    composeTestRule
        .onNodeWithText("Privacy Settings")
        .performClick()
    
    // Then - Should prompt for biometric auth
    composeTestRule
        .onNodeWithText("Biometric Authentication Required")
        .assertIsDisplayed()
    
    // When - Simulate successful auth
    simulateBiometricSuccess()
    
    // Then - Should show privacy settings
    composeTestRule
        .onNodeWithText("Privacy Settings")
        .assertIsDisplayed()
        
    composeTestRule
        .onNodeWithText("Data Retention")
        .assertIsDisplayed()
}
```

## Accessibility Tests

### 1. Screen Reader Compatibility
```kotlin
@Test
fun testTalkBackCompatibility() {
    // Given
    composeTestRule.setContent {
        FamilyGuardTheme {
            DashboardScreen(navController = rememberNavController())
        }
    }
    
    // Enable TalkBack simulation
    composeTestRule.onRoot().performTouchInput {
        // Double-tap and hold to simulate TalkBack
        down(Offset(100f, 100f))
        up()
        down(Offset(100f, 100f))
    }
    
    // Verify important elements have content descriptions
    composeTestRule
        .onNode(hasContentDescription("FamilyGuard Dashboard"))
        .assertExists()
        
    composeTestRule
        .onNode(hasContentDescription("Device status indicator"))
        .assertExists()
        
    composeTestRule
        .onNode(hasContentDescription("Battery level indicator"))
        .assertExists()
}

@Test
fun testHighContrastMode() {
    // Given - High contrast theme
    composeTestRule.setContent {
        FamilyGuardTheme(
            isHighContrast = true
        ) {
            DashboardScreen(navController = rememberNavController())
        }
    }
    
    // Verify sufficient contrast ratios
    composeTestRule
        .onNodeWithText("FamilyGuard Dashboard")
        .assert(hasContrastRatioAtLeast(4.5))
        
    composeTestRule
        .onNodeWithText("Device Status")
        .assert(hasContrastRatioAtLeast(4.5))
        
    // Verify focus indicators
    composeTestRule
        .onNodeWithTag("device_card")
        .performClick()
        
    composeTestRule
        .onNodeWithTag("device_card")
        .assert(hasFocusIndicator())
}
```

### 2. Dynamic Type Support
```kotlin
@Test
fun testLargeTextScaling() {
    // Test with different font scales
    val fontScales = listOf(1.0f, 1.5f, 2.0f, 3.0f)
    
    fontScales.forEach { fontScale ->
        // Given - Custom font scale
        composeTestRule.setContent {
            FamilyGuardTheme(
                fontScale = fontScale
            ) {
                DashboardScreen(navController = rememberNavController())
            }
        }
        
        // Verify no text truncation
        composeTestRule
            .onNodeWithText("FamilyGuard Dashboard")
            .assertIsDisplayed()
            .assert(hasNoTextTruncation())
            
        composeTestRule
            .onNodeWithText("Device Status")
            .assertIsDisplayed()
            .assert(hasNoTextTruncation())
        
        // Verify layout doesn't break
        composeTestRule
            .onRoot()
            .assert(hasNoOverlappingElements())
    }
}
```

## Performance Tests

### 1. UI Rendering Performance
```kotlin
@Test
fun testDashboardRenderingPerformance() {
    // Given - Large dataset
    val largeDeviceList = (1..100).map { index ->
        buildTestDevice().copy(childName = "Child $index")
    }
    
    testRepository.setDevices(largeDeviceList)
    
    // When - Measure rendering time
    val renderingTime = measureComposeRendering {
        composeTestRule.setContent {
            FamilyGuardTheme {
                DashboardScreen(navController = rememberNavController())
            }
        }
    }
    
    // Then - Should render within threshold
    assertThat(renderingTime).isLessThan(1000) // Less than 1 second
    
    // Verify smooth scrolling
    composeTestRule
        .onNodeWithTag("device_list")
        .performScrollToIndex(99)
        
    composeTestRule
        .onNodeWithText("Child 99")
        .assertIsDisplayed()
}

@Test
fun testMemoryUsageDuringNavigation() {
    // Given
    composeTestRule.setContent {
        FamilyGuardTheme {
            FamilyGuardApp()
        }
    }
    
    val initialMemory = getMemoryUsage()
    
    // When - Navigate through multiple screens
    repeat(10) { index ->
        composeTestRule
            .onNodeWithText("Test Child")
            .performClick()
            
        composeTestRule
            .onNodeWithText("Social")
            .performClick()
            
        composeTestRule.pressBack()
        composeTestRule.pressBack()
    }
    
    val finalMemory = getMemoryUsage()
    val memoryIncrease = finalMemory - initialMemory
    
    // Then - Memory increase should be minimal
    assertThat(memoryIncrease).isLessThan(50 * 1024 * 1024) // Less than 50MB
}
```

## Test Utilities

### 1. Custom Compose Assertions
```kotlin
fun hasContrastRatioAtLeast(minRatio: Double): SemanticsMatcher {
    return SemanticsMatcher("Contrast ratio at least $minRatio") { node ->
        val foreground = node.config.getOrNull(SemanticsProperties.Color)?.value
        val background = node.parent?.config?.getOrNull(SemanticsProperties.Color)?.value
        
        if (foreground != null && background != null) {
            val ratio = calculateContrastRatio(foreground, background)
            ratio >= minRatio
        } else {
            false
        }
    }
}

fun hasNoTextTruncation(): SemanticsMatcher {
    return SemanticsMatcher("No text truncation") { node ->
        val text = node.config.getOrNull(SemanticsProperties.Text)?.firstOrNull()?.text
        val bounds = node.boundsInRoot
        
        if (text != null && bounds.width > 0) {
            val textWidth = calculateTextWidth(text, node.fontSize)
            textWidth <= bounds.width
        } else {
            true
        }
    }
}

fun hasFocusIndicator(): SemanticsMatcher {
    return SemanticsMatcher("Has focus indicator") { node ->
        node.config.contains(SemanticsProperties.Focused) &&
        node.config.getOrNull(SemanticsProperties.Focused) == true
    }
}
```

### 2. Test Data Injection
```kotlin
object TestDataInjector {
    
    fun injectTestData(composeTestRule: ComposeTestRule) {
        // Inject test devices
        composeTestRule.setContent {
            FamilyGuardTheme {
                val viewModel: DashboardViewModel = hiltViewModel()
                
                LaunchedEffect(Unit) {
                    viewModel.setTestDevices(buildTestDeviceList())
                }
                
                DashboardScreen()
            }
        }
    }
    
    fun simulateNetworkError(composeTestRule: ComposeTestRule) {
        composeTestRule.setContent {
            FamilyGuardTheme {
                val viewModel: DashboardViewModel = hiltViewModel()
                
                LaunchedEffect(Unit) {
                    viewModel.simulateNetworkError()
                }
                
                DashboardScreen()
            }
        }
    }
    
    fun simulateEmptyState(composeTestRule: ComposeTestRule) {
        composeTestRule.setContent {
            FamilyGuardTheme {
                val viewModel: DashboardViewModel = hiltViewModel()
                
                LaunchedEffect(Unit) {
                    viewModel.setEmptyState()
                }
                
                DashboardScreen()
            }
        }
    }
}
```

## Test Execution Commands

### 1. Run UI Tests
```bash
# Run all UI tests
./gradlew connectedDebugAndroidTest \
  --tests "*ScreenTest"

# Run specific screen tests
./gradlew connectedDebugAndroidTest \
  --tests "*DashboardScreenTest"

./gradlew connectedDebugAndroidTest \
  --tests "*DeviceDetailScreenTest"

# Run with specific device
./gradlew connectedDebugAndroidTest \
  -Pandroid.testInstrumentationRunnerArguments.device=Pixel_6

# Run with coverage
./gradlew connectedDebugAndroidTest \
  -Pcoverage=true \
  -Pandroid.testInstrumentationRunnerArguments.coverageFile=build/outputs/code-coverage/connected/coverage.ec
```

### 2. Generate UI Test Reports
```bash
# Generate HTML report
./gradlew connectedDebugAndroidTest \
  -Pandroid.testInstrumentationRunnerArguments.reportDir=build/reports/ui-tests

# Generate screenshots
./gradlew connectedDebugAndroidTest \
  -Pandroid.testInstrumentationRunnerArguments.screenshotDir=build/outputs/screenshots

# Generate video recording
./gradlew connectedDebugAndroidTest \
  -Pandroid.testInstrumentationRunnerArguments.recordVideo=true
```

### 3. CI/CD Integration
```yaml
# GitHub Actions UI test job
ui-tests:
  runs-on: macos-latest
  steps:
    - name: Run UI Tests
      run: |
        ./gradlew connectedDebugAndroidTest \
          --tests "*ScreenTest" \
          --stacktrace
    
    - name: Upload Test Results
      uses: actions/upload-artifact@v3
      with:
        name: ui-test-results
        path: build/reports/androidTests/
    
    - name: Upload Screenshots
      uses: actions/upload-artifact@v3
      with:
        name: ui-test-screenshots
        path: build/outputs/screenshots/
```

## Test Maintenance

### 1. UI Test Checklist
- [ ] All interactive elements have tests
- [ ] Navigation flows are tested
- [ ] Error states are handled
- [ ] Loading states are shown
- [ ] Empty states are displayed
- [ ] Accessibility features work
- [ ] Performance meets requirements
- [ ] Different screen sizes supported

### 2. Test Data Management
```kotlin
@After
fun cleanupUITest() {
    // Clear test data
    TestDatabase.clearAll()
    
    // Remove test files
    TestFileUtils.cleanupTestFiles()
    
    // Reset preferences
    TestPreferences.reset()
    
    // Clear navigation state
    TestNavController.reset()
}
```

### 3. Test Documentation
- Document test scenarios and expected behavior
- Include screenshots for visual verification
- Note any device-specific requirements
- Document accessibility test results
- Include performance benchmarks

## Conclusion

These UI tests ensure the FamilyGuard Parent app provides a reliable, accessible, and performant user experience. Regular execution of these tests, especially with different device configurations and accessibility settings, ensures the app remains user-friendly and functional across all supported environments.

The tests cover:
- **Functional correctness**: All features work as expected
- **User experience**: Smooth navigation and interactions
- **Accessibility**: Support for all users
- **Performance**: Fast rendering and low memory usage
- **Error handling**: Graceful degradation when issues occur

---

**Test Coverage Targets**:
- Dashboard: 95% UI test coverage
- Device Details: 90% UI test coverage  
- Settings: 85% UI test coverage
- Navigation: 100% flow coverage
- Accessibility: 100% WCAG 2.1 AA compliance

**Required Test Devices**:
- Phone: Pixel 6 (API 33), Galaxy S22 (API 31)
- Tablet: Pixel Tablet (API 33), Galaxy Tab S8 (API 31)
- Foldable: Pixel Fold (API 33)