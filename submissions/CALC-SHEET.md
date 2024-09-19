# Code Analysis for Calc-Sheet

## Overview

Tech Stacks:

* Frontend: `React`
* Backend: `Express (Node.js)`

## Frontend Analysis

## `App.tsx` Analysis

This is the entry point of the project, and handles the top level of  states for routing as well as spreadsheet clients.

### Routing System

This demo uses `window.location.href` to parse URL path and `getDocumentNameFromWindow()` function to dynamically load components/page.

* Simple, but in modern design would much prefer to use **[`React Router`](https://reactrouter.com/en/main)** for more efficient and scalable routing management.

* The default route is the `LoginPageComponent`, which is rendered when the URL is `/documents`, and it navigates users to the appropriate document or page after a successful login.

```typescript
function getDocumentNameFromWindow() {
  const href = window.location.href;
  // remove  the protocol 
  const protoEnd = href.indexOf('//');
  // find the beginning of the path
  const pathStart = href.indexOf('/', protoEnd + 2);
  if (pathStart < 0) {
    // there is no path
    return '';
  }
  // get the first part of the path
  const docEnd = href.indexOf('/', pathStart + 1);
  if (docEnd < 0) {
    // there is no other slash
    return href.substring(pathStart + 1);
  }
  // there is a slash
  return href.substring(pathStart + 1, docEnd);
}
```

### State Management

```typescript
const [documentName, setDocumentName] = useState(getDocumentNameFromWindow());
useEffect(() => {
  if (window.location.href) {
    setDocumentName(getDocumentNameFromWindow());
  }
}, [getDocumentNameFromWindow]);
```

* Uses React `useState` and `useEffect` at the top level to pass on state down to the components, such as the `documentName`.

* The `useEffect` hook then watches the `documentName` as dependencies and updates component/page when documentName gets changed

* `LoginPageComponent` receives `spreadSheetClient` as a prop to handle user interactions and backend communication related to authentication and document access.

* `SpreadSheet` component receives `documentName` and `spreadSheetClient` as props to manage the specific document and backend communication.

## `LoginPageComponent` Analysis

This component is responsible for handling the actual login process and displaying a list of documents for the user to select and edit once logged in.

### State Management within Login

* `userName`: Stores the logged-in username, which is initialized from `window.sessionStorage`. This ensures that the username is persisted across page reloads.

* `documents`: An array of document names fetched from the server, stored and displayed to the user.

### Polling Mechanism

* The use of `setInterval` in the useEffect hook is used to poll for available documents from the `spreadSheetClient` every 50 milliseconds. When documents are fetched, they are stored in the documents state.

```typescript
useEffect(() => {
  const interval = setInterval(() => {
    const sheets = spreadSheetClient.getSheets();
    if (sheets.length > 0) {
      setDocuments(sheets);
    }
  }, 50);
  return () => clearInterval(interval);
});
```

### Login and Logout

* **Login Mechanism:** The login is handled via an input field where the user enters their name and presses "Enter". The username is stored in `sessionStorage` and the `spreadSheetClient` object is updated with the username.

* **Logout Mechanism:** When the user logs out, the username is cleared from `sessionStorage` and the page reloads to reset the state.

```typescript
function getUserLogin() {
  return <div>
    <input
      type="text"
      placeholder="User name"
      defaultValue={userName}
      onKeyDown={(event) => {
        if (event.key === 'Enter') {
          let userName = (event.target as HTMLInputElement).value;
          window.sessionStorage.setItem('userName', userName);
          setUserName(userName);
          spreadSheetClient.userName = userName;
        }
      }} />
  </div>
}
```

### Document Management

* **Document Selection**: Once logged in, the user is presented with a list of documents fetched from the spreadSheetClient. Each document has an associated "Edit" button, which triggers the loading of the selected document by updating the URL and reloading the page.



## Backend Analysis

## Integration Analysis

