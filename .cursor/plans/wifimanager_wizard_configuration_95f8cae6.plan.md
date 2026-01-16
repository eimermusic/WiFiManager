# WiFiManager Step-by-Step Configuration Wizard

## Overview

Transform the configuration portal into a wizard-style flow with:

- **Step 1**: WiFi credentials entry with real-time verification (keeping portal open)
- **Step 2**: Custom parameters (if they exist) with verification callback
- Navigation: Back, Skip, and Next buttons
- Final success page with option to exit

## Architecture Changes

### 1. Wizard State Management

- Add wizard state enum: `WIZARD_STEP_NONE`, `WIZARD_STEP_WIFI`, `WIZARD_STEP_PARAMS`, `WIZARD_STEP_COMPLETE`
- Add state tracking variable: `_wizardStep` in `WiFiManager` class
- Add flag: `_wizardMode` to enable/disable wizard mode (default: false for backward compatibility)

### 2. WiFi Verification (Step 1)

**File: `WiFiManager.cpp`**

- Modify `handleWifiSave()` to:
- Verify WiFi credentials using `WIFI_AP_STA` mode
- Use short timeout (3-5 seconds) for connection attempt
- Keep AP active during verification
- Show verification result page (success/failure)
- Store verification result in state
- Do NOT close portal on success - allow progression to Step 2

**New function: `verifyWiFiCredentials(String ssid, String pass)`**

- Switch to `WIFI_AP_STA` mode
- Attempt connection with timeout
- Return connection status
- Disconnect STA but keep AP running
- Restore to AP-only mode

**File: `WiFiManager.h`**

- Add method: `setWizardMode(bool enable)` to enable wizard mode
- Add method: `setWifiVerificationTimeout(unsigned long timeout)` (default 5000ms)

### 3. Custom Parameter Verification (Step 2)

**File: `WiFiManager.h`**

- Add callback type: `std::function<bool()> _verifyParamscallback`
- Add method: `setVerifyParamsCallback(std::function<bool()> func)`
- Callback should return `true` if parameters are valid, `false` otherwise

**File: `WiFiManager.cpp`**

- Modify `handleParamSave()` to:
- Save parameters first
- Call verification callback if set
- Show verification result page
- Allow progression to completion or back to Step 2

### 4. Navigation System

**New routes:**

- `/wizard/step1` - WiFi credentials step
- `/wizard/step2` - Custom parameters step  
- `/wizard/next` - Proceed to next step
- `/wizard/back` - Go to previous step
- `/wizard/skip` - Skip current step
- `/wizard/complete` - Final success page

**File: `WiFiManager.cpp`**

- Add handlers:
- `handleWizardStep1()` - Show WiFi form
- `handleWizardStep2()` - Show params form
- `handleWizardNext()` - Navigation handler
- `handleWizardBack()` - Navigation handler
- `handleWizardSkip()` - Skip handler
- `handleWizardComplete()` - Success page

### 5. HTML Templates

**Files: `wm_strings_en.h` (and other language files)**

- Add wizard-specific HTML templates:
- `HTTP_WIZARD_STEP1` - WiFi form with "Next" button
- `HTTP_WIZARD_STEP2` - Params form with "Next" and "Back" buttons
- `HTTP_WIZARD_VERIFYING` - Verification in progress message
- `HTTP_WIZARD_WIFI_SUCCESS` - WiFi verification success
- `HTTP_WIZARD_WIFI_FAILED` - WiFi verification failed
- `HTTP_WIZARD_PARAMS_SUCCESS` - Params verification success
- `HTTP_WIZARD_PARAMS_FAILED` - Params verification failed
- `HTTP_WIZARD_COMPLETE` - Final success page with "Exit" button
- `HTTP_WIZARD_BACK_BTN` - Back button
- `HTTP_WIZARD_SKIP_BTN` - Skip button
- `HTTP_WIZARD_NEXT_BTN` - Next button

### 6. Modified Existing Handlers

**File: `WiFiManager.cpp`**

- `handleRoot()`: If wizard mode enabled, redirect to `/wizard/step1` instead of showing menu
- `handleWifi()`: If wizard mode, show Step 1 form with wizard styling
- `handleParam()`: If wizard mode, show Step 2 form with wizard styling

### 7. State Persistence

- Store entered WiFi credentials temporarily during wizard flow
- Store custom parameters temporarily
- Only save permanently after final success
- Clear temporary state on wizard exit/abort

## Implementation Details

### WiFi Verification Flow

1. User enters SSID/password in Step 1
2. On submit, show "Verifying..." message
3. Switch to `WIFI_AP_STA` mode
4. Attempt connection with timeout
5. Show result (success with green checkmark, or failure with error message)
6. If success: Show "Next" button to proceed to Step 2
7. If failure: Show "Back" button to retry, keep credentials in form
8. Disconnect STA, restore AP-only mode

### Custom Parameters Verification Flow

1. User enters custom parameters in Step 2
2. On submit, save parameters
3. Call verification callback if set
4. Show result based on callback return value
5. If success: Show "Complete" button
6. If failure: Show error and "Back" button to retry

### Navigation Logic

- **Back**: Go to previous step, preserve entered data
- **Skip**: 
- In Step 1: Skip to Step 2 (if params exist) or Complete
- In Step 2: Skip to Complete
- **Next**: Validate current step, then proceed
- **Exit**: Close portal and return control to main app

## Files to Modify

1. **WiFiManager.h**

- Add wizard state enum and variables
- Add wizard mode flag
- Add verification callback
- Add new public methods

2. **WiFiManager.cpp**

- Implement wizard state management
- Implement WiFi verification function
- Modify existing handlers for wizard mode
- Add new wizard handlers
- Add navigation logic

3. **wm_strings_en.h** (and other language files)

- Add wizard HTML templates
- Add wizard-specific strings

## Backward Compatibility

- Wizard mode is **opt-in** via `setWizardMode(true)`
- Default behavior remains unchanged (single-page form)
- Existing callbacks and functionality preserved

## Testing Considerations

- Test with iPhone/Android devices connected to portal during verification
- Verify AP stays active during STA connection attempts
- Test timeout behavior with slow/fast networks
- Test navigation between steps
- Test skip/back functionality
- Test with and without custom parameters§