# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

**react-payment-inputs** is a zero-dependency React library that provides hooks and components for payment card input fields. It handles automatic formatting, validation, and focus management for card numbers, expiry dates, CVCs, and ZIP codes. The library supports both hooks (`usePaymentInputs`) and render props (`PaymentInputsContainer`) patterns.

## Build and Development Commands

### Building
- `yarn build` or `npm run build` - Build the library using Rollup. Creates ES modules (`es/`), CommonJS (`lib/`), and UMD (`umd/`) bundles
- `yarn clean` or `npm run clean` - Remove all build artifacts and proxy directories

### Linting
- `yarn lint` or `npm run lint` - Run ESLint on the `src/` directory

### Storybook
- `yarn storybook` or `npm run storybook` - Start Storybook development server on port 6006
- `yarn build-storybook` or `npm run build-storybook` - Build static Storybook to `docs/` directory

### Publishing
- `yarn prepublishOnly` - Automatically runs before publishing; creates proxy directories and builds
- `yarn postpublish` - Automatically runs after publishing; cleans up build artifacts

**Note:** This project does not have automated tests. The Storybook examples in `stories/index.stories.js` serve as the primary testing/verification mechanism.

## Architecture

### Core Exports (src/index.js)
The library exports three main items:
1. **`usePaymentInputs`** - React Hook for payment inputs (primary API)
2. **`PaymentInputsContainer`** - Render props wrapper around `usePaymentInputs`
3. **`PaymentInputsWrapper`** - Styled wrapper component (requires styled-components as peer dependency)

### Hook Architecture (src/usePaymentInputs.js)

The `usePaymentInputs` hook is the heart of the library (~530 lines). It manages:

**State:**
- `touchedInputs` - Track which fields have been interacted with
- `erroredInputs` - Store validation errors for each field
- `cardType` - Detected card type based on card number
- `focused` - Currently focused field
- Field refs for cardNumber, expiryDate, cvc, and zip inputs

**Prop Getters Pattern:**
The hook returns "prop getter" functions that must be spread onto input elements:
- `getCardNumberProps(overrideProps)` - Returns props for card number input
- `getExpiryDateProps(overrideProps)` - Returns props for expiry date input
- `getCVCProps(overrideProps)` - Returns props for CVC input
- `getZIPProps(overrideProps)` - Returns props for ZIP input
- `getCardImageProps({ images })` - Returns props for card type SVG image
- `wrapperProps` - Props to pass to `PaymentInputsWrapper`

**Important:** Event handlers (onChange, onBlur, etc.) must be passed INSIDE the prop getter functions to avoid overriding the library's internal handlers. Example: `getCardNumberProps({ onChange: myHandler })` not `{...getCardNumberProps(), onChange: myHandler}`.

**Auto-focus Behavior:**
The hook automatically focuses the next field when a field is completed (controlled by `autoFocus` option, default `true`). The focus chain is: cardNumber → expiryDate → cvc → zip.

**Validation Flow:**
1. On change: Format input → Validate → Update error state → Auto-focus if valid
2. On blur: Mark field as touched → Show error if exists
3. Cursor repositioning: After formatting card numbers, the hook repositions the cursor to handle space insertions

### Utilities (src/utils/)

**cardTypes.js** - Card type definitions and detection
- `CARD_TYPES` - Array of card configurations including Visa, Mastercard, Amex, Discover, JCB, UnionPay, Maestro, Elo, Hipercard, Troy, and Cuban card types (BPA, BANDEC)
- Each card type has: `displayName`, `type`, `format` (regex), `startPattern` (regex for detection), `gaps` (spacing positions), `lengths` (valid lengths), `code` (CVC/CVV info)
- `getCardTypeByValue(value)` - Detects card type from card number
- `getCardTypeByType(type)` - Gets card type by type string

**validator.js** - Input validation logic
- `getCardNumberError()` - Validates card number using Luhn algorithm and card type length
- `getExpiryDateError()` - Validates MM/YY format and ensures date is not in past
- `getCVCError()` - Validates CVC length based on card type (3 or 4 digits)
- `getZIPError()` - Basic ZIP validation (non-empty)
- `validateLuhn(cardNumber)` - Luhn checksum algorithm implementation
- `hasCardNumberReachedMaxLength()` - Checks if card number is complete
- `isNumeric(e)` - Validates keypress is numeric

**formatter.js** - Input formatting logic
- `formatCardNumber(cardNumber)` - Adds spaces based on card type format (e.g., "1234 5678 9012 3456")
- `formatExpiry(event)` - Formats expiry as "MM / YY" with intelligent insertion (e.g., "1" → "01 / ")

### Styled Wrapper (src/PaymentInputsWrapper.js)

Uses styled-components to create a single-line payment input field combining all inputs with card image. Structure:
- `FieldWrapper` - Outer container
- `InputWrapper` - Inner container with border/focus states
- `ErrorText` - Error message display

Supports custom styling via `styles` prop with schema for: `fieldWrapper`, `inputWrapper`, `input` (with sub-keys for each field type), and `errorText`. Accepts both styled-components `css` and plain objects.

### Build System

**Rollup Configuration (rollup.config.js):**
- Builds ES modules, CommonJS, and UMD bundles
- Uses `rollup-plugin-proxy-directories` to create proxy entry points for individual exports (enables `import { usePaymentInputs } from 'react-payment-inputs/usePaymentInputs'`)
- External peer dependencies: `react`, `styled-components`, `prop-types`

**Babel Configuration (.babelrc.js):**
- Presets: `@babel/preset-env`, `@babel/preset-react`
- Plugins: class-properties, object-rest-spread, dynamic-import, export-namespace-from

### Proxy Directories
The build creates proxy directories at the root (`PaymentInputsContainer/`, `PaymentInputsWrapper/`, `usePaymentInputs/`, etc.) that enable direct imports. These are generated by `scripts/create-proxies.js` and removed by `scripts/remove-proxies.js`.

## Key Patterns and Conventions

### Event Handler Merging
The library merges user-provided event handlers with internal handlers. User handlers are called first, then internal handlers. This is why handlers must be passed inside prop getters.

### Ref Forwarding
The library uses `ref` by default but supports custom ref keys via the `refKey` parameter in prop getters. Example: `getCardNumberProps({ refKey: 'inputRef' })` for libraries like Fannypack.

### Custom Validators
Users can provide custom validators for additional business logic:
- `cardNumberValidator({ cardNumber, cardType, errorMessages })` - Called after Luhn validation passes
- `expiryValidator({ expiryDate: { month, year }, errorMessages })` - Called after format validation passes
- `cvcValidator({ cvc, cardType, errorMessages })` - Called after length validation passes

### Error Messages
All error messages can be customized via the `errorMessages` option object. Available keys: `emptyCardNumber`, `invalidCardNumber`, `emptyExpiryDate`, `monthOutOfRange`, `yearOutOfRange`, `dateOutOfRange`, `invalidExpiryDate`, `emptyCVC`, `invalidCVC`, `emptyZIP`.

## Integration Notes

### styled-components Requirement
`PaymentInputsWrapper` requires styled-components >=4.0.0 as a peer dependency. The basic hook can be used without styled-components.

### Card Images
Card images are exported from `react-payment-inputs/images` as SVG elements (React components). Supported cards: amex, dinersclub, discover, hipercard, jcb, mastercard, troy, unionpay, visa, and a placeholder.

### Form Library Integration
The library integrates with form libraries (Formik, React Final Form, etc.) by passing form library handlers into the prop getters. The `meta.erroredInputs` object should be mapped to the form library's error state.

### No TypeScript
This project is JavaScript-only. There are no TypeScript definitions in the repository.

## File Structure

- `src/` - Source code
  - `usePaymentInputs.js` - Main hook (530+ lines, most complex file)
  - `PaymentInputsContainer.js` - Render props wrapper (6 lines, thin wrapper)
  - `PaymentInputsWrapper.js` - Styled wrapper component
  - `utils/` - Utility modules
    - `cardTypes.js` - Card type definitions
    - `validator.js` - Validation logic
    - `formatter.js` - Formatting logic
  - `images/` - Card type SVG images
- `stories/` - Storybook examples (serves as test suite)
- `scripts/` - Build scripts for proxy directory management
- `.storybook/` - Storybook configuration

## Important Implementation Details

### Cursor Positioning
The hook uses `requestAnimationFrame` to reposition the cursor after card number formatting adds spaces. Without this, the cursor would jump to the end of the input.

### Touch Detection
The `isTouched` state becomes true when focus leaves all payment inputs (checked via `document.activeElement.tagName !== 'INPUT'`). Individual field touch state is tracked separately in `touchedInputs`.

### Card Type Detection
Card types are detected on every card number change by testing the value against each card type's `startPattern` regex. The first match is used.

### Expiry Date Formatting Edge Cases
The expiry formatter handles several edge cases:
- Single digit month (2-9) → prepend 0
- Months >12 → split and reformat (e.g., "15" → "01 / 5")
- "1/" input → auto-format to "01 / "
- Delete handling to move between month/year sections
