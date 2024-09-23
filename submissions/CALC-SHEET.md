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

* `getDocumentNameFromWindow` is a function that extracts the document name from the current URL. It parses the path portion of the URL, finds the document name, and returns it.
  If there is not enough information in the URL (e.g. the path portion is missing), it returns an empty string.
* `resetURL(documentName)` is used to update the browser address bar when the URL changes. It removes the last part of the URL and adds the new document name to the URL. It uses `window.history.pushState`to update the URL and reload the page.

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
* `SpreadSheet` component receives `documentName` and `spreadSheetClient` as props to manage the specific document and backend communication. Communicate with the spreadsheet server through the `spreadSheetClient` instance and provide an error handling callback function `displayErrorMessage`.

## `LoginPageComponent` Analysis

This component is responsible for handling the actual login process and displaying a list of documents for the user to select and edit once logged in.

### State Management within Login

* `userName`: Stores the logged-in username, which is initialized from `window.sessionStorage`. This ensures that the username is persisted across page reloads.
* `documents`: An array of document names fetched from the server, stored and displayed to the user. (obtained from the server via `SpreadSheetClient`).

### Polling Mechanism

* The use of `setInterval` in the useEffect hook is used to poll for available documents from the `spreadSheetClient` every 50 milliseconds. When documents are fetched, they are stored in the documents state. When the component is unmounted, clear the timer to prevent memory leaks by function of `clearInterval(interval)`. This way of checking whether the document is loaded every 50 milliseconds may cause performance issues. You can consider introducing a callback mechanism in `spreadSheetClient` or increase the delay to reduce frequent status checks.

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
* The `checkUserName` function checks if userName is empty. If it is empty, a warning message pops up to ask the user to enter the username. But in fact, I didn't see any warning meesage it when testing. Another flaw is that the username is saved only when the user presses the `Enter` key. There is no prompt on the interface to ask to press enter. Better interface interaction with the user is needed. Also, there is currently no error handling mechanism. If `getSheets()` fails, the interface will not feedback error message.
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

* The `SpreadSheetClient` class primarily implements the **`Facade pattern`**, providing a simplified interface for handling complex interactions between the frontend and backend. It abstracts away the details of network requests and document management, allowing React components to interact with the backend through easy-to-use methods.
* It also incorporates elements of the **`Proxy pattern`**, acting as an intermediary that manages communication with the server and document updates. While not explicitly a **`singleton`**, it shares some characteristics by centralizing the management of user and document state through a **single instance**.

## Backend Analysis

### 1. RESTful API Analysis:

#### **CRUD Operations:**

The backend exposes various endpoints to perform CRUD operations on documents and cells within a spreadsheet system.

- **Create:**

  - **`POST /documents/create/:name`**: Creates a new document with the specified name and a user (passed in the request body). The document is initialized with a grid of cells and stored on the server.
- **Read:**

  - **`GET /documents`**: Fetches a list of all available document names.
  - **`GET /documents/:name`**: Retrieves the contents of the document in JSON format for the given name.
  - **`GET /document/formula/string/:name`**: Returns the current formula for a specific cell within a document.
  - **`GET /document/result/string/:name`**: Returns the computed result of a formula from the document.
  - **`GET /document/editstatus/:name`**: Retrieves the edit status of the current document.
- **Update:**

  - **`PUT /documents/:name`**: Updates a document or creates it if it does not exist. The request body should include `userName`.
  - **`PUT /document/addtoken/:name`**: Adds a token to the formula for a specific document and user.
  - **`PUT /document/addcell/:name`**: Adds a cell reference to the formula for a specific document and user.
  - **`PUT /document/cell/edit/:name`**: Grants a user the ability to edit a specific cell in a document.
  - **`PUT /document/cell/view/:name`**: Grants a user view access to a specific cell in a document.
  - **`PUT /document/removetoken/:name`**: Removes the last token from the formula in a document.
  - **`PUT /document/clear/formula/:name`**: Clears the formula for a document.
- **Delete:**

  - **`POST /documents/reset`**: Resets all documents (used for testing or clearing data).

#### **Validation and Processing:**

- The backend validates requests by checking the existence of required parameters like `userName` and `cell`.
- If a document or user is not found, appropriate error responses (e.g., 404 Not Found or 400 Bad Request) are returned.
- For example, the `PUT /document/cell/edit/:name` route checks if the document exists and whether a user has edit access before proceeding.

### 2. Real-Time Communication:

While WebSockets or real-time communication mechanisms like WebRTC are not directly implemented in the given code, a **polling mechanism** mimics real-time interaction:

- **_timedFetch** method in the client fetches the document data from the server at short intervals (every 0.1 seconds).
- This ensures that changes made by other users are reflected frequently, simulating a near real-time experience.

### 3. User Management:

#### **Authentication**

- The system uses the `userName` in requests to identify users and manage their access to documents. The user's state (viewing or editing) is tracked using the `ContributingUser` class.

#### **Role Management:**

User privileges are determined by whether they are allowed to edit or only view a document.

- **Viewers**: Users who have read-only access to a cell/document.
- **Editors**: Users who are actively editing a cell. Only one user can edit a specific cell at any given time, and the `_cellsBeingEdited` map ensures exclusive access.

#### **Security of Sensitive Data:**

Sensitive data security mechanisms (e.g., encryption of passwords or tokens) are not present in the provided code. User session handling and token-based security like JWT are not implemented in this setup.

### 4. Middleware and Error Handling:

#### **Middleware:**

- **CORS Middleware:** `app.use(cors())` is used to handle Cross-Origin Resource Sharing, ensuring that client applications from different origins can access the API.
- **Body Parser Middleware:** `bodyParser.json()` is used to parse incoming request bodies in JSON format. This ensures that request data is easily accessible via `req.body`.
- **Request Logging Middleware:** The backend logs each incoming request using a custom middleware (`app.use((req, res, next) => { if (debug) console.log(req.method, req.url); next(); }`). This helps in debugging API traffic.

#### **Error Handling:**

- If a required parameter is missing (e.g., `userName` in a request), the server responds with a 400 status and an error message like `'userName is required'`.
- The code has basic checks to ensure documents or users exist before proceeding. For example, when accessing a document or requesting cell editing rights, the server checks if the document exists and whether the cell is being edited by another user.
- Error handling in API responses is simple and clear, returning appropriate status codes such as 404 for not found documents and 400 for bad requests.

### 5. Explore Design Patterns

This project uesd client-server architecture, several design patterns are employed to ensure efficient and organized interaction between components. Notably, the **Singleton**, **Observer**, and **Model-View-Controller (MVC)** patterns are utilized.

#### 1. Singleton Pattern

**Purpose:** Ensures that a class only has one instance and provides a global point of access to it.

**Implementation in Code:**

In the `DocumentHolder` class, a Singleton pattern is implemented to manage the collection of documents. This ensures that only one instance of `DocumentHolder` exists, providing a centralized point for document management.

```javascript
class DocumentHolder {
  constructor() {
    if (!DocumentHolder.instance) {
      this._documents = new Map();
      DocumentHolder.instance = this;
    }
    return DocumentHolder.instance;
  }
  // ... other methods
}
```

#### 2. Observer Pattern

**Purpose:** Defines a one-to-many dependency between objects, allowing observers to receive updates when the subject changes.

**Implementation in Code:**

While the code doesn't explicitly implement the Observer pattern, the client-side polling mechanism acts similarly by observing the server for changes. The `SpreadSheetClient` class periodically fetches updates from the server, ensuring that the client’s state reflects the latest data.

```javascript
class SpreadSheetClient {
  constructor() {
    this._timedFetch();
  }

  _timedFetch() {
    setTimeout(() => {
      this.getDocument();
      this._timedFetch();
    }, 100);
  }
  // ... other methods
}
```

#### 3. Model-View-Controller (MVC) Pattern

**Purpose:** Separates an application into three main components: Model (data), View (user interface), and Controller (business logic), promoting organized and maintainable code.

**Implementation in Code:**

The backend architecture follows the MVC pattern:

- **Model:** Represented by classes like `SheetMemory` and `Cell`, which manage the data structure of the spreadsheet.
- **View:** The client-side application that renders the spreadsheet to the user.
- **Controller:** The `SpreadSheetController` class acts as the controller, handling user interactions and updating the model accordingly.

```javascript
class SpreadSheetController {
  constructor() {
    this._memory = new SheetMemory();
    this._calculationManager = new CalculationManager();
  }
  // ... other methods
}
```

By employing these design patterns, the system ensures a structured, efficient, and maintainable client-server interaction, facilitating seamless collaboration and data consistency.

### `DocumentServer.ts` Analysis

This file defines the server-side functionality for managing documents using an Express server. The server provides a range of RESTful endpoints that interact with the `DocumentHolder` class, which manages the spreadsheet documents in memory.

#### Key Functionalities:

1. **Document Management:**

   - The server manages documents stored in the `DocumentHolder` class. Each document is represented as a `SpreadSheetController` object, allowing multiple users to interact with it concurrently.
2. **API Routes:**

   - The server defines several API routes to interact with documents, including:
     - `GET /documents`: Retrieves the list of available documents.
     - `PUT /documents/:name`: Creates or fetches a specific document based on the name provided.
     - `PUT /document/cell/edit/:name`: Requests edit access to a specific cell within a document.
     - `PUT /document/addtoken/:name`: Adds a token to the formula of the current cell.
     - `PUT /document/removetoken/:name`: Removes the last token from the formula of the current cell.
     - `PUT /document/clear/formula/:name`: Clears the current formula of the selected cell.
3. **Error Handling:**

   - Each route includes basic error handling to ensure requests are correctly processed. For example, if a document does not exist, a 404 response is returned.
4. **Logging and Debugging:**

   - The server includes debug logging, which can be toggled on or off to help developers trace the flow of requests and identify potential issues during development.

### `DocumentHolder.ts` Analysis

`DocumentHolder` manages the collection of documents (`SpreadSheetController` instances). It handles the creation, modification, and retrieval of spreadsheet documents.

#### Key Functionalities:

1. **Document Storage:**

   - Documents are stored in a Map data structure, providing quick access and efficient management of multiple spreadsheets.
2. **Document Operations:**

   - The class provides methods to create new documents, save documents to the file system, and load documents from existing JSON data.
3. **Concurrency Control:**

   - It includes mechanisms to manage edit and view access requests from multiple users, ensuring data consistency and preventing conflicts.
4. **Error Management:**

   - The class includes robust error handling to manage invalid operations, such as accessing non-existent documents or circular dependencies between cells.

### `SpreadSheetController.ts` Analysis

The `SpreadSheetController` class is the core logic for managing spreadsheet operations. It handles interactions with the `SheetMemory`, which stores the state of each spreadsheet.

#### Key Functionalities:

1. **Cell Operations:**

   - The class manages adding, removing, and editing tokens within a cell's formula. It uses a `FormulaEvaluator` to compute the results of these formulas dynamically.
2. **Concurrency Handling:**

   - The controller manages multiple users editing and viewing cells, ensuring that only one user can edit a cell at any given time.
3. **Error Checking:**

   - It includes error checking for common issues such as circular dependencies, divide-by-zero errors, and invalid formulas.

### `FormulaEvaluator.ts` Analysis

`FormulaEvaluator` is responsible for parsing and evaluating cell formulas. It implements a recursive descent parser to handle arithmetic operations, unary and postfix operators, and cell references.

#### Key Functionalities:

1. **Arithmetic Parsing:**

   - The class supports parsing of basic arithmetic operations (`+`, `-`, `*`, `/`), unary operators (negative values), and various mathematical functions (e.g., `sin`, `cos`).
2. **Error Management:**

   - Errors such as invalid formulas, divide-by-zero, and missing parentheses are identified and returned as part of the evaluation process.
3. **Integration with Sheet Memory:**

   - The evaluator fetches values directly from the `SheetMemory` class, ensuring that all calculations are based on the latest state of the spreadsheet.

## Integration Analysis

### Frontend-Backend Interaction

The Calc-Sheet project integrates the frontend and backend through a RESTful API, facilitating the seamless exchange of data and operations between the user interface and server-side logic.

#### Key Integration Points:

1. **Document Management:**

   - The frontend uses the `SpreadSheetClient` class to communicate with backend endpoints, such as fetching document data, editing cells, and updating formulas.
2. **Real-time Updates:**

   - The polling mechanism in `LoginPageComponent` ensures that the list of available documents is kept up-to-date in near real-time, enhancing user experience.
3. **Concurrency Control:**

   - The backend handles concurrent access and editing requests, ensuring data integrity and preventing conflicts during collaborative editing sessions.
4. **Error Handling:**

   - Errors encountered during operations (e.g., invalid formulas, circular dependencies) are communicated back to the frontend and displayed to the user, enabling corrective actions.

### Recommended Improvements

1. **Routing Optimization:**

   - Replace the manual URL parsing in `App.tsx` with a robust routing library like `React Router` for better routing management.
2. **State Management Enhancement:**

   - Consider integrating a state management solution like `Redux` or `Context API` to manage global state more effectively, particularly as the application scales.
3. **Improved Error Feedback:**

   - Enhance the user feedback mechanism for errors, moving from simple alerts to more informative and user-friendly UI notifications.
4. **Performance Optimization:**

   - Optimize backend performance by reducing the frequency of polling intervals or implementing WebSockets for real-time updates.
5. **Security Enhancements:**

   - Implement authentication and authorization mechanisms to secure document access and ensure that only authorized users can edit or view specific cells.

## Conclusion

The Calc-Sheet project effectively combines React and Express to create a collaborative spreadsheet editing experience. The design showcases solid use of design patterns, concurrency management, and real-time user interaction. With some enhancements to routing, state management, and security, the project can be further improved to provide a robust and scalable solution.
