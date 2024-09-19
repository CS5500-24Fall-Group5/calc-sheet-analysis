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

* **Document Selection**: Once logged in, the user is presented with a list of documents fetched from the `spreadSheetClient`. Each document has an associated "Edit" button, which triggers the loading of the selected document by updating the URL and reloading the page.

## `SpreadSheet` Analysis

### State Management within SpreadSheet

The SpreadSheet component uses several useState hooks to manage the state of the spreadsheet:

* `formulaString`: Holds the current formula being entered or edited.
* `resultString`: Holds the result of the formula calculation.
* `cells`: Stores the display values of the spreadsheet cells.
* `statusString`: Manages the status of the current editing session (whether editing or not).
* `currentCell`: Tracks the currently selected cell.
* `currentlyEditing`: Indicates whether a cell is being edited or not.
* `userName`: Holds the user’s name, fetched from sessionStorage.
* `serverSelected`: Allows the user to switch between different servers (default is "localhost").

### Event Handlers

The component contains several event handlers to manage user interactions:

* `onCommandButtonClick`: Handles the logic when command buttons (such as "edit", "clear", etc.) are clicked. It processes the button action and updates the spreadsheet accordingly.
* `onButtonClick`: Handles the number or parenthesis buttons during formula editing. When a button is clicked, it adds the corresponding value to the formula.
* `onCellClick`: Manages cell clicks. If the user is currently editing, it adds the cell's value to the formula. Otherwise, it updates the view to show the clicked cell.

### Interaction with `spreadSheetClient`

The `spreadSheetClient` is heavily utilized throughout the component for managing state and communicating with the backend:

* **Data fetching:** The client fetches the current formula, result, status, and cell values from the backend.
* **User and document management:** The client manages the user's name and the current document.
* **Command execution:** The client processes commands like editing, clearing, and token management, ensuring the backend reflects the changes.

### Return to Login Page

The component includes a function, `returnToLoginPage`, that allows the user to return to the login page by updating the URL and reloading the page. This resets the session and allows the user to select a new document.

```typescript
function returnToLoginPage() {
  spreadSheetClient.documentName = documentName;
  const href = window.location.href;
  const index = href.lastIndexOf('/');
  let newURL = href.substring(0, index);
  newURL = newURL + "/documents";
  window.history.pushState({}, '', newURL);
  window.location.reload();
}
```

## `SpreadSheetClient` Class Analysis

The `SpreadSheetClient` class is responsible for facilitating communication between the frontend (React components) and the backend server. It handles all the CRUD operations for the spreadsheet data, manages the document state, and updates the UI based on interactions with the backend.

## Backend Analysis

## Integration Analysis
