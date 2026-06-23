# Auth — Technical Documentation

## Overview
The `auth.js` module serves as the central identity management layer for the application. It wraps the Firebase Authentication SDK to provide a consistent, simplified interface for user sign-in, registration, and session lifecycle management. It is designed to act as a source of truth for the user's authentication state across the entire web application.

## Architecture
This module sits at the infrastructure/service layer of the application. 
- **Dependencies:** Relies on `firebase/app` and `firebase/auth`.
- **Dependents:** Used by UI components or routing logic that require session persistence, user identification, or access control.
- **Role:** It acts as a singleton mediator, abstracting Firebase-specific complexities and providing a centralized mechanism for navigation redirection based on auth state.

## Design Principles
- **Singleton Pattern:** By exporting a single instance (`authService`), the module ensures that the application maintains a unified session state throughout the browser lifecycle.
- **Observer Pattern:** The `onAuthStateChanged` and `notifyAuthStateListeners` implementation allows external modules to "subscribe" to session changes without direct coupling to Firebase logic.
- **Fail-Safe Encapsulation:** All asynchronous operations are wrapped in `try/catch` blocks, returning normalized response objects (`{ success, user/error }`) rather than throwing raw exceptions, promoting predictable API consumption.

## API Reference

### `AuthService` Class
A singleton class handling all authentication operations.

#### Methods
*   **`signInWithEmail(email, password)`**
    *   `email` (string): User email address.
    *   `password` (string): User password.
    *   *Returns:* `Promise<{ success: boolean, user?: Object, error?: string }>`
*   **`signUpWithEmail(email, password)`**
    *   *Returns:* Same structure as `signInWithEmail`. Creates a new user in Firebase.
*   **`signInWithGoogle()`**
    *   *Returns:* `Promise<{ success: boolean, user?: Object, error?: string }>`
*   **`signOut()`**
    *   *Returns:* `Promise<{ success: boolean, error?: string }>`
*   **`getCurrentUser()`**
    *   *Returns:* `User|null` (Firebase User object or null if unauthenticated).
*   **`isAuthenticated()`**
    *   *Returns:* `boolean` (True if user is signed in).
*   **`onAuthStateChanged(callback)`**
    *   `callback` (function): Executed when the auth state changes. Receives the user object as an argument.

## Internal Logic
1.  **Initialization:** Upon instantiation, the module triggers `initializeAuthState`. This binds an `onAuthStateChanged` observer to Firebase.
2.  **Navigation Guard:** The internal observer inspects `window.location.pathname`.
    *   If a user is logged in, it prevents access to `login`, `signup`, or `landing` pages (redirecting to `project.html`).
    *   If a user is logged out, it forces a redirect to `login.html` if the current page is not an entry/auth page.
3.  **State Synchronization:** The `currentUser` property is kept in sync with the Firebase backend at all times, ensuring `isAuthenticated()` is always accurate.

## Data Flow
1.  **Authentication:** Data enters via method arguments (email/password/Google). 
2.  **External Sync:** Firebase validates credentials and returns a `userCredential`.
3.  **Transformation:** The module catches the result and transforms it into a standard `success/error` payload.
4.  **Broadcast:** The `onAuthStateChanged` listener triggers, updating the internal `currentUser` and notifying all registered application listeners.

## Error Handling & Edge Cases
*   **Async Errors:** Firebase exceptions (e.g., "wrong password", "network error") are caught and mapped to the `error` string in the return object.
*   **Listener Failures:** The `notifyAuthStateListeners` method includes a `try/catch` block to ensure that if one subscriber throws an error, the subsequent subscribers still receive the update.
*   **Redirection Loops:** The logic uses explicit path checking (`.includes()`) to determine redirects, preventing infinite loops during state shifts.

## Usage Example

### Basic Auth Check
```javascript
import authService from './auth.js';

if (authService.isAuthenticated()) {
    console.log("Welcome back:", authService.getCurrentUser().email);
} else {
    console.log("Please log in.");
}
```

### Handling Sign-In
```javascript
import authService from './auth.js';

const handleLogin = async (email, password) => {
    const result = await authService.signInWithEmail(email, password);
    if (result.success) {
        console.log("Logged in!");
    } else {
        alert(`Login failed: ${result.error}`);
    }
};
```