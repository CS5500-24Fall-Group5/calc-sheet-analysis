# Code Analysis for Calc-Sheet

## Overview

Tech Stacks:

* Frontend: `React`
* Backend: `Express (Node.js)`

## Frontend Analysis

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


## Backend Analysis

## Integration Analysis

