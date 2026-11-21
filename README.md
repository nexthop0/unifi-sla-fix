# IMPORTANT DISCLOSURE: AI-GENERATED TROUBLESHOOTING AND REPORT

> **Please read before reviewing the technical findings.** I am a **network engineer, not a web developer**, and I have no working knowledge of JavaScript, React, React Hook Form, or browser-based application debugging. **The troubleshooting strategy, Developer Tools commands/code, technical interpretation, and this written report were generated and directed by OpenAI ChatGPT (AI).**
>
> My role was to operate my own UDM-SE and browser, execute the steps and commands ChatGPT supplied, and provide the observed outputs. I personally confirmed that the UniFi mobile app could change the WAN SLA, that the web interface initially would not apply changes, and that the web interface successfully saved the WAN SLA change after toggling Automatic Speed Test on and then off.
>
> **Please treat this as an AI-assisted diagnostic report, not an independently audited web-development analysis.** The console output and observed workaround are reported as encountered, but ChatGPT's interpretations—including the underlying product defect or how the invalid form value originated—should be independently verified by Ubiquiti engineering. I cannot independently validate the JavaScript analysis or code.

---

# UniFi Network 11.1.104: WAN SLA “Apply Changes” silently blocked by `speedTest.enabled` validation

## Environment

- **Gateway:** UniFi Dream Machine Special Edition (UDM-SE)
- **UniFi OS:** 6.0.17
- **UniFi Network:** 11.1.104
- **Connection:** Local web interface to console; UniFi remote/cloud access disabled
- **Affected:** WAN SLA changes in the UniFi Network web UI, across multiple browsers
- **Unaffected:** The UniFi mobile app successfully changed the same WAN SLA settings

## Symptoms and history

On the Internet settings page, editing a WAN SLA value revealed **Apply Changes**, but clicking it produced no visible response, no error message, and no new save request in DevTools **Network → Fetch/XHR**. The same change succeeded from the mobile app.

The issue persisted after full recovery/factory reset, reinstallation/upgrading of UniFi Network, and restoration of a UniFi Network application backup. Because the backup was restored, that sequence alone does not rule out restored configuration state.

## Investigation (DevTools commands immediately before their observed results)

All JavaScript below was executed in Chrome/Edge DevTools on the **local** UniFi Network page. These were diagnostic reads, breakpoint commands, or event monitors—not direct requests to the gateway API or database edits. The commands inspect browser-internal form state, which is not a stable public API.

### 1. Apply button was enabled and clickable

With **Apply Changes** selected using the Elements inspector (`$0`), run:

```javascript
const el = $0.closest('button, [role="button"]') || $0;

({
  tag: el.tagName,
  disabled: el.disabled,
  ariaDisabled: el.getAttribute('aria-disabled'),
  pointerEvents: getComputedStyle(el).pointerEvents,
  outerHTML: el.outerHTML.slice(0, 700)
});
```

**Observed result (HTML abbreviated only for readability):**

```javascript
{
  tag: "BUTTON",
  disabled: false,
  ariaDisabled: null,
  pointerEvents: "auto",
  outerHTML: '<button type="submit" class="...">' +
             '<span class="...">Apply Changes</span>...</button>'
}
```

### 2. Native HTML form validation passed

With the same button selected, run:

```javascript
const btn = $0.closest('button');
const form = btn.form;

({
  formFound: !!form,
  valid: form ? form.checkValidity() : null,
  invalidFields: form
    ? [...form.elements]
        .filter(el => el.willValidate && !el.validity.valid)
        .map(el => ({
          tag: el.tagName,
          name: el.name,
          type: el.type,
          message: el.validationMessage
        }))
    : []
});
```

**Observed result:**

```javascript
{
  formFound: true,
  valid: true,
  invalidFields: []
}
```

Native HTML validation was not the problem, but application-level validation had not yet been evaluated.

### 3. `click` and `submit` events fired, but no save request followed

With the button still selected as `$0`, run:

```javascript
monitorEvents($0, 'click');
monitorEvents($0.form, 'submit');
```

Then click **Apply Changes** once while observing **Console**, and separately observe **Network → Fetch/XHR**.

**Observed result:** Chrome logged a trusted `click` and a `submit` event targeting `form#internet-general-form`; the submit event had `defaultPrevented: true`. No new Fetch/XHR save request appeared. Preventing the native submit is normal for a JavaScript-managed form, so `defaultPrevented` alone is not an error.

The Console also repeatedly logged `Missing queryFn` in `9616.0d547ae80775ed82.js`, with `refetch` / `onFocus` stack frames. **Its causal relationship to the Apply failure was not established.**

### 4. Breakpoint identified the React form submission handler

Select the form element in the browser using the Apply button, then run:

```javascript
const formEl = document.getElementById('internet-general-form');
const reactKey = Object.keys(formEl).find(
  key => key.startsWith('__reactProps$')
);
const submitHandler = reactKey
  ? formEl[reactKey].onSubmit
  : undefined;

console.log({
  reactPropsFound: !!reactKey,
  submitHandlerFound: typeof submitHandler === 'function'
});

if (typeof submitHandler === 'function') {
  debug(submitHandler);
}
```

**Observed result:** Clicking Apply paused in `test-config.09050aba405417c4a45e.js` at the minified expression:

```javascript
e.handleSubmit(Fe)(t)
```

This is consistent with a React Hook Form `handleSubmit` path. While **paused on that frame**, capture the controller reference in the Console:

```javascript
window.__slaForm = e;
```

Then resume with **F8**. This captures the existing browser-side controller; it does not set a UniFi configuration field.

### 5. Application-level form validation failed on `speedTest`

After the attempted submission, run:

```javascript
const s = window.__slaForm?.control?._formState;

({
  stateFound: !!s,
  submitCount: s?.submitCount,
  isSubmitted: s?.isSubmitted,
  isSubmitting: s?.isSubmitting,
  isValid: s?.isValid,
  errors: Object.entries(s?.errors ?? {}).map(
    ([field, error]) => ({
      field,
      type: error?.type,
      message: error?.message
    })
  )
});
```

**Observed result:**

```javascript
{
  stateFound: true,
  submitCount: 3,
  isSubmitted: true,
  isSubmitting: false,
  isValid: false,
  errors: [
    { field: "speedTest" }
  ]
}
```

The error was nested, so the top-level `speedTest` error did not have its own `type` or `message`.

### 6. Exact nested validation error: `speedTest.enabled`

Run:

```javascript
const err = window.__slaForm?.control?._formState?.errors?.speedTest;

({
  errorPresent: !!err,
  type: err?.type ?? null,
  message: err?.message ?? null,
  keys: Object.keys(err ?? {}).filter(k => k !== "ref"),
  children: Object.fromEntries(
    Object.entries(err ?? {})
      .filter(([k, v]) =>
        k !== "ref" && v && typeof v === "object"
      )
      .map(([k, v]) => [k, {
        type: v?.type ?? null,
        message: v?.message ?? null,
        keys: Object.keys(v).filter(k => k !== "ref")
      }])
  )
});
```

**Observed result:**

```javascript
{
  errorPresent: true,
  type: null,
  message: null,
  keys: ["enabled"],
  children: {
    enabled: {
      type: "invalid_type",
      message: "COMMON_VALIDATION_VALUE_REQUIRED{::}{}",
      keys: ["message", "type"]
    }
  }
}
```

**Key finding:** The Internet settings web form refused submission due to `speedTest.enabled`, an Automatic Speed Test field unrelated to the changed WAN SLA value.

### 7. Form value existed as a property but was `undefined`

Run:

```javascript
const value = window.__slaForm?.getValues?.("speedTest");

({
  speedTestExists: value !== undefined && value !== null,
  availableFields: value && typeof value === "object"
    ? Object.keys(value)
    : [],
  enabledValue: value?.enabled,
  enabledType: typeof value?.enabled
});
```

**Observed result:**

```javascript
{
  speedTestExists: true,
  availableFields: [
    "_id", "key", "site_id", "speed_test_mode",
    "cron_expr", "dayInMonth", "occurence", "startTime",
    "weekDays", "mode", "wanList", "enabled"
  ],
  enabledType: "undefined"
}
```

`enabledValue` was omitted from the object displayed by DevTools in the captured result because it was `undefined`. The important point is that the **field name was present, but its value was not initialized**.

## Successful workaround and verification

The **Automatic Speed Test** checkbox in **Settings → Internet** was originally **unchecked**.

1. Check **Automatic Speed Test**.
2. Uncheck it again (restoring the original visible setting).
3. **Do not refresh** before inspecting form state and saving.

Run:

```javascript
({
  enabled: window.__slaForm.getValues("speedTest.enabled"),
  type: typeof window.__slaForm.getValues("speedTest.enabled")
});
```

**Observed result after the toggle:**

```javascript
{
  enabled: false,
  type: "boolean"
}
```

4. Change the intended WAN SLA value.
5. Click **Apply Changes**.

**Observed result:** **The WAN SLA setting saved successfully from the web interface.** No manual API request, database modification, factory reset, or cloud access was needed for the workaround.

## Conclusions and requested engineering investigation

**Confirmed:** The web interface initially represented `speedTest.enabled` as `undefined`. Application-level validation rejected it with `invalid_type` / `COMMON_VALIDATION_VALUE_REQUIRED{::}{}` and prevented an unrelated WAN SLA edit from reaching the save request. Toggling Automatic Speed Test on and back off initialized the form field to Boolean `false`; Apply then worked.

**Not established:** Whether `undefined` originated in persisted data, restored/migrated configuration, or a web-frontend initialization/mapping defect. Also not yet verified: whether the workaround persists after reloading the page and whether the `Missing queryFn` errors are related.

**Requested investigation:**

- Why is `speedTest.enabled` not initialized as a Boolean when Automatic Speed Test is unchecked?
- Does the Internet form in Network 11.1.104 fail to normalize restored or migrated speed-test configuration?
- Why does a validation failure in Automatic Speed Test silently block WAN SLA changes without showing the user which field needs attention?
- Can form defaults and error presentation be corrected so unrelated Internet settings can be saved reliably?

## Optional DevTools cleanup

Stop event monitoring if the original selected elements still exist:

```javascript
unmonitorEvents(formEl, 'submit');
// For the button originally selected as $0:
// unmonitorEvents($0, 'click');
```

Remove any debugging breakpoints in **Sources → Breakpoints** and uncheck **Sources → Event Listener Breakpoints → Control → submit**, if enabled. Reload the page to discard temporary JavaScript references. The DevTools Console can then be cleared. This cleanup does not revert saved UniFi settings.
