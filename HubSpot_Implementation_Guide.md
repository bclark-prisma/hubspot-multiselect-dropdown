# HubSpot Multiselect Implementation Guide

**Complete step-by-step guide for implementing the FormData Injection method to convert HubSpot checkbox fields into searchable dropdowns.**

## 🎯 Overview

This guide explains how to implement the **FormData Injection** method that successfully converts HubSpot "Multiple Checkboxes" fields into searchable dropdowns while maintaining proper data mapping to Ticket properties.

## 📋 Prerequisites

### HubSpot Configuration Requirements

1. **Form Setup**:
   - Form must have "Automatic ticket creation" enabled
   - Form must be connected to a Conversations Inbox or Help Desk
   - Ticket Pipeline and Status must be configured in Inbox settings

2. **Property Setup**:
   - Target field must be a **Ticket property** (not Contact property)
   - Field type must be "Multiple checkboxes"
   - Property must have "Use in forms, pop-up forms, and bots" enabled

3. **Permissions**:
   - Super Admin or Account Access permissions required
   - Access to form editor and property settings

## 🔍 Step 1: Identify Your Target Field

### Finding the Field Name

1. **Inspect the Form**: Open your HubSpot form in a browser
2. **Open Developer Tools**: Press F12
3. **Find Checkboxes**: Look for checkbox inputs in the DOM
4. **Extract Field Name**: Note the `name` attribute pattern

**Example:**
```html
<input name="0-5/target_languages_multi" value="Russian [RU], Russian [RU]" type="checkbox">
<input name="0-5/target_languages_multi" value="Finnish [FI], Finnish [FI]" type="checkbox">
```

**Key Pattern**: `input[name*="target_languages_multi"]`

### Understanding HubSpot's Field Naming

HubSpot uses this naming pattern:
- `{form-id}/{property-internal-name}`
- Example: `0-5/target_languages_multi`
- The part after `/` is your property's internal name

## 🛠️ Step 2: Customize the Script

### Required Changes

1. **Update Field Selector** (Line ~53):
```javascript
// CHANGE THIS:
const languageCheckboxes = group.querySelectorAll('input[name*="target_languages_multi"]');

// TO YOUR FIELD:
const languageCheckboxes = group.querySelectorAll('input[name*="YOUR_PROPERTY_NAME"]');
```

2. **Update Field Label** (Line ~86):
```javascript
// CHANGE THIS:
const labelText = fieldLabel ? fieldLabel.textContent : 'Target Language(s)*';

// TO YOUR LABEL:
const labelText = fieldLabel ? fieldLabel.textContent : 'Your Field Label*';
```

3. **Update Console Messages** (Optional):
```javascript
// Update log messages to reflect your field name for easier debugging
console.log("✅ Found", checkboxes.length, "YOUR_FIELD_NAME checkboxes");
```

### Script Locations to Update

| Line | What to Change | Example |
|------|----------------|---------|
| ~53 | Field selector | `input[name*="your_field"]` |
| ~60 | Debug message | `"Found X your_field checkboxes"` |
| ~86 | Label text | `"Your Field Label*"` |
| ~406 | Success message | `"your_field multiselect created!"` |

## 📤 Step 3: Deploy the Script

### Option A: Footer HTML (Recommended)

1. **Navigate**: Go to your HubSpot page editor
2. **Advanced Options**: Click "Advanced Options" 
3. **Footer HTML**: Paste the entire script content
4. **Publish**: Save and publish the page

### Option B: External JavaScript File

1. **Host Script**: Upload script to your hosting/CDN
2. **Add Script Tag**: 
```html
<script src="https://your-domain.com/hubspot-multiselect.js"></script>
```

### Option C: Inline on Page

1. **Edit Page**: Open page in HubSpot editor
2. **Add HTML Module**: Insert Rich Text or HTML module
3. **Paste Script**: Include the complete script with `<script>` tags

## 🔍 Step 4: Understanding the Mechanism

### How FormData Injection Works

1. **DOM Extraction**: Script extracts checkbox options and values before removal
2. **Complete Removal**: Original checkbox group is completely removed from DOM
3. **UI Creation**: Custom searchable dropdown is created using HubSpot's CSS classes
4. **State Tracking**: Selected values are tracked in a JavaScript Set
5. **FormData Override**: The FormData constructor is overridden during submission
6. **Data Injection**: Selected values are injected as a semicolon-delimited string

### Key Technical Components

#### Field Detection Algorithm
```javascript
// Searches all checkbox groups for your specific field
checkboxGroups.forEach(group => {
  const checkboxes = group.querySelectorAll('input[name*="YOUR_FIELD"]');
  if (checkboxes.length > 0) {
    targetGroup = group; // Found the target field
  }
});
```

#### Value Extraction and Mapping
```javascript
// Creates a map of values to checkbox elements before removal
const checkboxMap = new Map();
checkboxes.forEach(cb => checkboxMap.set(cb.value, cb));

// Extracts display labels from values
const labelText = originalCheckbox.value.includes(',') 
  ? originalCheckbox.value.split(',')[0].trim()  // "Russian [RU]" from "Russian [RU], Russian [RU]"
  : originalCheckbox.value;
```

#### FormData Interception
```javascript
// Override FormData constructor globally
const originalFormData = window.FormData;
window.FormData = function(formElement, submitter) {
  const formData = new originalFormData(formElement, submitter);
  
  if (isOurFormSubmission && formElement === form) {
    // Remove any existing entries for our field
    // Add our semicolon-delimited values
    const semicolonDelimitedValue = Array.from(selectedValues).join(';');
    formData.append(fieldName, semicolonDelimitedValue);
  }
  
  return formData;
};
```

#### Data Format Requirements

HubSpot expects multiple checkbox values as a single semicolon-delimited string:

**Correct Format**: `"Russian [RU], Russian [RU];Finnish [FI], Finnish [FI];Spanish [ES], Spanish [ES]"`

**Incorrect Formats**:
- Array: `["Russian [RU]", "Finnish [FI]"]` ❌
- Comma-separated: `"Russian [RU], Finnish [FI]"` ❌  
- Individual entries: Multiple FormData entries ❌

## 🧪 Step 5: Testing & Verification

### Testing Checklist

- [ ] **Visual Verification**: Original checkboxes disappear, dropdown appears
- [ ] **UI Testing**: Dropdown opens/closes, search works, selections display
- [ ] **Console Monitoring**: Check for successful injection messages
- [ ] **Form Submission**: Form submits without errors
- [ ] **Data Verification**: Values appear in HubSpot Ticket record

### Expected Console Output

```bash
🚀 Setting up enhanced multiselect with FormData injection...
✅ Found 384 your_field checkboxes
🔍 Field name detected: 0-5/your_field_name
🗑️ Removed original checkbox group from DOM
📋 Form found, setting up FormData interception
✅ HubSpot multiselect with FormData injection created!

# During form submission:
🚀 === FORM SUBMISSION INTERCEPTED ===
🎯 Will inject values: ["Value1", "Value2", "Value3"]
💉 Injecting language selection into FormData
✅ Injected into FormData: 0-5/your_field_name = Value1;Value2;Value3
```

### Troubleshooting Console Messages

| Message | Meaning | Action |
|---------|---------|--------|
| `⚠️ No checkbox group found` | Field selector doesn't match | Update field selector |
| `❌ Could not find checkbox for value` | Value mapping failed | Check checkbox value format |
| `🗑️ Removed original checkbox group` | DOM removal successful | ✅ Normal operation |
| `💉 Injecting language selection` | FormData override working | ✅ Normal operation |

## 🚨 Common Issues & Solutions

### Issue 1: Script Doesn't Run
**Symptoms**: No console messages, original checkboxes remain
**Causes**: 
- Script not loaded properly
- JavaScript errors preventing execution
- HubSpot form not detected

**Solutions**:
- Check browser console for JavaScript errors
- Verify script is in correct location (Footer HTML)
- Ensure HubSpot form loads before script runs

### Issue 2: Field Not Found
**Symptoms**: "No checkbox group found" message
**Causes**:
- Incorrect field selector
- Form hasn't loaded when script runs
- Field name changed in HubSpot

**Solutions**:
- Inspect DOM to verify exact field name
- Update selector to match actual name attribute
- Add delay or use HubSpot form events

### Issue 3: Data Not Submitted
**Symptoms**: Dropdown works but values don't appear in HubSpot
**Causes**:
- FormData injection not working
- Wrong field name in injection
- Form submission bypassing override

**Solutions**:
- Check console for injection messages
- Verify field name matches exactly
- Ensure FormData override is active during submission

### Issue 4: UI Doesn't Match HubSpot Style
**Symptoms**: Dropdown looks different from native HubSpot fields
**Causes**:
- Missing HubSpot CSS classes
- Custom CSS interfering
- Theme overrides

**Solutions**:
- Verify all `hsfc-*` class names are correct
- Check for CSS conflicts in browser inspector
- Add `!important` to critical styles if needed

## 🔄 Adapting for Different Field Types

### For Different Data Formats

If your checkboxes have different value formats, update the label extraction:

```javascript
// Current (for "Russian [RU], Russian [RU]" format):
const labelText = originalCheckbox.value.includes(',') 
  ? originalCheckbox.value.split(',')[0].trim()
  : originalCheckbox.value;

// For simple values ("Russian", "Finnish"):
const labelText = originalCheckbox.value;

// For custom formats ("RU: Russian Language"):
const labelText = originalCheckbox.value.split(':')[1]?.trim() || originalCheckbox.value;
```

### For Required Fields

If your field is required, ensure the asterisk appears:

```javascript
// Check if original field was required
const isRequired = Array.from(checkboxes).some(cb => cb.hasAttribute('required'));

// Add asterisk to label if required
const labelText = fieldLabel ? fieldLabel.textContent : 'Your Field Label' + (isRequired ? '*' : '');
```

### For Different UI Requirements

**Hide Search Box**:
```javascript
// Comment out search container creation
// searchContainer.appendChild(searchInput);
```

**Change Maximum Height**:
```css
.hsfc-DropdownOptions__List {
  max-height: 200px !important; /* Change from 175px */
}
```

**Disable Multi-Select** (Convert to Single Select):
```javascript
// In option click handler, clear other selections first
optionsList.addEventListener('click', function(e) {
  if (e.target.classList.contains('hsfc-DropdownOptions__List__ListItem')) {
    // Clear all other selections first
    optionsList.querySelectorAll('.hsfc-DropdownOptions__List__ListItem--multi-selected')
      .forEach(item => {
        item.classList.remove('hsfc-DropdownOptions__List__ListItem--multi-selected');
        selectedValues.clear();
      });
    
    // Then add the new selection
    // ... rest of selection logic
  }
});
```

## 📚 Advanced Customization

### Adding Validation

```javascript
// Add validation before form submission
form.addEventListener('submit', function(e) {
  if (selectedValues.size === 0 && isRequired) {
    e.preventDefault();
    alert('Please select at least one option');
    return false;
  }
  // ... rest of submission logic
});
```

### Adding Loading States

```javascript
// Show loading during form submission
form.addEventListener('submit', function(e) {
  displayInput.value = 'Submitting...';
  displayInput.disabled = true;
});
```

### Custom Event Dispatching

```javascript
// Dispatch custom events for integration with other scripts
function updateDisplayText() {
  // ... existing code ...
  
  // Dispatch custom event
  const customEvent = new CustomEvent('hubspot-multiselect-change', {
    detail: { selectedValues: Array.from(selectedValues) }
  });
  document.dispatchEvent(customEvent);
}

// Listen for the event elsewhere
document.addEventListener('hubspot-multiselect-change', function(e) {
  console.log('Selection changed:', e.detail.selectedValues);
});
```

## 🎯 Best Practices

### Performance Optimization

1. **Minimal DOM Queries**: Cache selectors and reuse elements
2. **Event Delegation**: Use single event listeners where possible
3. **Debounced Search**: Add debouncing for search input if needed
4. **Lazy Loading**: Only create dropdown options when first opened

### Maintainability

1. **Configuration Object**: Move all customizable values to a config object
2. **Modular Functions**: Break down large functions into smaller, testable pieces
3. **Error Handling**: Add try-catch blocks around critical operations
4. **Documentation**: Comment complex logic and HubSpot-specific requirements

### Accessibility

1. **ARIA Labels**: Add appropriate ARIA attributes
2. **Keyboard Navigation**: Ensure all interactions work with keyboard
3. **Screen Reader Support**: Test with screen readers
4. **Focus Management**: Maintain logical focus order

## 📝 Maintenance Checklist

### Regular Checks

- [ ] **HubSpot Updates**: Monitor for HubSpot form changes that might affect the script
- [ ] **Browser Compatibility**: Test in different browsers periodically  
- [ ] **Performance**: Monitor console for any new errors or warnings
- [ ] **Data Integrity**: Regularly verify data is still reaching Ticket records correctly

### When HubSpot Updates

1. **Test Functionality**: Verify dropdown still works after HubSpot updates
2. **Check Console**: Look for new errors or deprecation warnings
3. **Update Selectors**: HubSpot might change CSS class names or structure
4. **Verify Data Flow**: Ensure FormData injection still works correctly

## 🆘 Support & Troubleshooting

### Debug Mode

Add this debug flag for enhanced logging:

```javascript
const DEBUG_MODE = true; // Set to false for production

function debugLog(message, data = null) {
  if (DEBUG_MODE) {
    console.log(`[HubSpot Multiselect Debug] ${message}`, data);
  }
}
```

### Contact Points

For issues specific to:
- **HubSpot Configuration**: Check HubSpot's support documentation
- **Form Setup**: Verify Ticket property configuration
- **Script Errors**: Use browser developer tools
- **Data Mapping**: Test with native checkboxes first to isolate issues

---

## 📄 Conclusion

This FormData Injection method provides a robust, reliable solution for enhancing HubSpot's checkbox fields. The key insight is that instead of fighting HubSpot's form processing, we bypass it entirely by injecting data directly into the FormData stream.

The solution is:
- **Production-ready**: Successfully tested with 384 options
- **Maintainable**: Clean, documented code structure  
- **Extensible**: Easy to adapt for different fields and requirements
- **Reliable**: Bypasses the checkbox synchronization issues that plague other approaches

Follow this guide carefully, and you'll have a working searchable dropdown that properly submits data to your HubSpot Ticket properties. 