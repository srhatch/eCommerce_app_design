# eCommerce_app_design
## An explanation of design choices for an eCommerce application using React
I have been building an eCommerce type application and want to share some of the approaches to designing and programming that I’ve used. The general idea behind this app is that a user’s role is set so they are buying or selling goods with a fixed disposition. This means that a user would not be both buying and selling within the market. The intent is also to promote local trade, so searching isn’t done for large distances. Examples of these types of markets are catering/food prep, landscaping services, or artwork (especially commissioned). Essentially, the market is composed of sellers that have some kind of specialized service or product instead of a market in which users buy and sell items they have not produced themselves. Both buyers and sellers can post listings. The application has standard features like user accounts, posting listings, saving listings, contacting users, and searching for listings.
<br><br>
The following examples are abstractions of what is actually written in the app. They are meant to illustrate the design patterns I’ve used to address certain problems. As such, they do not depict full implementations that would potentially include other code, like aria attributes or the complete range of event handlers needed for proper UI responsiveness. They also are not bound to a particular directory structure, so the import statements are vague. If the same file is referenced multiple times, or in multiple examples, it will be designated with a name that will stay consistent throughout the entire document. Assume this is a single page application with some type of client-side routing framework like React Router or Remix. 

### Table of Contents
  - [Explanation of React Router](#a-brief-explanation-of-react-router) 
  - [Client side user data cache](#client-side-user-data-cache)
  - [Loading](#loading)
  - [.then() vs async/await](#then-vs-asyncawait)
  - [Code splitting](#code-splitting)
  - [&lt;ConfirmAction&gt; component](#confirmaction-component)
  - [Modular CSS](#modular-css)

### A brief explanation of React Router:
React Router uses client-side navigation, which means that when navigation actions happen in the browser a GET request will return only a page’s dynamic data instead of returning a full HTML file. Intra-app navigations happen when either a form is submitted, or a link is clicked. With React Router, a single page is defined by a component, which is mapped to a URL. When that URL is present in the address bar the component will mount to the DOM. Functions can be connected to these components to run either when the component is mounted (“loader”), or when a form is submitted (“action”). A loader function is used to make HTTP requests that fill the page with data. Action functions can be used to submit Fetch requests with form data.
### Client-side user data cache
I wanted to implement a quick search feature so that signed in users would not have to enter redundant information, mainly their location and search market, every time they wanted to search for listings. This requires information to be present in the browser at the time of the search. It might not be a terrible idea to store this data in the session object on the server and then access it as the search request comes in, but this could become memory intensive and may not scale well if many users end up using the application at once. There are also other components, aside from the quick search, in the client where this kind of information can be used.
<br><br>
The next potential solution would be one of the two WebStorage APIs. This could be a viable option because the data being stored isn’t sensitive and is easily found on a user’s public profile. sessionStorage would not be a good idea because it’s not shared across browser tabs, so localStorage would have to be used.
<br><br>
I didn’t end up using WebStorage mainly for two reasons. First, there is a risk of the data being accessed and changed. Second, using localStorage would require managing expiration time, repopulation of cleared data (for example if a user clears cookies), and revalidation of expired data. In both of these scenarios the functionality of the app would break.
<br><br>
I decided to use a context to store and pass down this data. The context value is an object containing the state (userCache) and the setState function (setUserCache). userCache is an object containing the user data. setUserCache is used by components like login and settings so the UI can be properly updated.
<br><br>
A component (called AppContext) is used to wrap the rest of the application to pass the context, and to store the context values.
<br><br>
app_context.js:
<br>
![alt text](https://github.com/user-attachments/assets/ab4eea48-e6e5-4168-815a-3dc19548bb2a)
<br><br>
Again, assuming a React Router or Remix type structure, <Outlet/> would contain the rest of the components in the app. Whenever there is a full-page refresh, AppContext will make an HTTP request with getUserCacheData() and then update its state. getUserCacheData() is a promise returning function that can be defined in a data service layer. This layer would also be responsible for handling any errors coming back from the server, so the useEffect() setup function is only concerned with updating userCache.
<br><br>
Notes:
<br>
The &lt;h1&gt; elements describing <main> are all defined in what would be the current page as designated by <Outlet/>. This allows flexibility and removes the need to write a <main> element in each of the page components.
<br><br>
A brief explanation of the useEffect syntax can be found [below](#then-vs-asyncawait).
<br><br>
The quick search component could be implemented as follows.
<br><br>
quick_search.js:
<br>
![alt text](https://github.com/user-attachments/assets/abf6709f-5f67-423c-aeee-83cf57e3adaf)
<br><br>
The input element is not controlled and there is no processing done on the value, so I’m using a ref here, but this could easily be changed to have the value held in state. Now, when the search is submitted, the user’s city, state, and searchMarket are all taken from context. searchMarket is the opposite of a user’s market disposition, so for example if a user is a buyer, their search market would be selling, and the search would request listings of users who are selling products.
<br><br>
Some components might need the ability to update the UI, as would be the case with a login component. setUserCache() can be used to do this and is accessed through the context. In the following example, login() is also a promise returning function as described above, so the HTTP request logic and the component (UI) logic can be separated.
<br><br>
login.js:
<br>
![alt text](https://github.com/user-attachments/assets/5c6178c1-db8d-452f-a041-3b8b7e46af24)
<br><br>
Logging out follows a similar process: set the userCacheContext to an empty object and navigate.
### Loading
With this setup there ended up being two types of conditions where a loading UI was needed. One is a full page reload with a logged in user. Certain elements of the UI are dependent on the data in the userCache, so when a logged in user refreshes the page there is potential for those elements, with asynchronous operations, to load after the rest of the UI. This results in a jarring visual experience. To address this, I conditionally rendered a <Loading> component if the userCache is null, which is its initial state. Once the userCache is populated with an object (either empty or filled with user data) the rest of the app will render.
<br><br>
app_context.js:
<br>
![alt text](https://github.com/user-attachments/assets/a59c6ceb-d0db-4a19-9bfd-e2c59f558ad3)
<br><br>
The other condition happens when a component fetches data as it’s being rendered. This can be triggered either by submitting a form, as would be the case for a search or settings update, or by clicking a link. To address this, I created a &lt;LoadingWrapper&gt; component that conditionally applies a className, based on the current loading state, to a &lt;div&gt; container that reduces the opacity of the page. The loading state is an enumerated string value (“idle” or “loading”) obtained either from a custom loading context for POST requests, or from React Router’s useNavigation() hook for GET requests. The &lt;LoadingWrapper&gt; renders its children inside of this &lt;div&gt;, which allows the wrapper to be used as granularly as desired. 
<br><br>
loading_wrapper.js:
<br>
![alt text](https://github.com/user-attachments/assets/c661104b-adc6-4067-987a-b142b811206a)
<br><br>
The CSS ruleset could look like this:<br>
![alt text](https://github.com/user-attachments/assets/264d69b9-fa7c-40ca-b659-44d6595131bf)
<br><br>
This is imported into the AppContext to wrap the main content of the app. 
<br><br>
app_context.js:
<br>
![alt text](https://github.com/user-attachments/assets/f1e0ec73-7bc0-4be9-940b-3e37b75d338c)
<br><br>
Keep in mind that the loading state from useNavigation() is changed when any loader function is run. This means the loading UI style will be applied to any child components of <LoadingWrapper> even if the loading state was changed by an unrelated component.
<br><br>
The custom loading state context mimics useNavigation’s behavior. Global context holds the navigation state value in AppContext. The state is then changed in whichever function is called when a form is submitted. &lt;LoadingWrapper&gt; reads this value and determines whether to add the className to the &lt;div&gt; wrapper.
<br><br>
app_context.js:
<br>
![alt text](https://github.com/user-attachments/assets/bdd1e091-797f-4b9a-a1ed-9db5b886ebbf)
<br><br>
I used a global, custom &lt;Form&gt; component, mainly to make form inputs controlled and to keep track of their values in a centralized location, so I put the loading state changing logic in the form’s submit handler. The component takes a submit handler as one of its props and this handler will take three arguments, the last of which is a function that will set the loading state back to “idle”. This gives more flexibility to the submit handler, which can implement its logic however it needs and then can run this callback when its finished.
<br><br>
Here is a very stripped-down version of the &lt;Form&gt; component just to show how this would be hooked up.
<br><br>
form.js:
<br>
![alt text](https://github.com/user-attachments/assets/370b2e95-2241-42d9-ac1b-678e0e26b028)
<br><br>
Passing inputValues and setLoadingState to handleFormSubmit leaves room for it to be moved to another location if necessary.
<br><br>
Another very stripped-down version of a component that might use &lt;Form&gt;.
<br><br>
new_listing.js:
<br>
![alt text](https://github.com/user-attachments/assets/68ac1f76-a0a2-44d7-bf42-2caa1a6b9753)
<br><br>
This solution was tweaked a bit to work with React Router/Remix by using the built-in useNavigation(), but the same callback delineated state setting could be used with a custom &lt;Link&gt; component for GET requests.
<br><br>
It is possible to structure this with a more typical React Router setup by using an action function to run on form submission, but I think this setup is less extensible. The values would be accessible as a FormData object so this would ideally be converted to an object before any validation code is run. The action function would be responsible for checking errors and if there were any it would then return an error object. This would be accessed in the component with the useActionData() hook, but to keep the error signaling responsive (disappearing when the user types or changes an input) it would need to be set into state. useEffect would check for changes to the return value of useActionData() and update a state variable if there were any. This ends up requiring a bit more processing just to keep the code idiomatic with React Router, so I decided to customize the form submission logic.
### .then() vs async/await
For components that call a promise returning function from the useEffect setup function, it can be more readable to use then() instead of async/await.
<br><br>
![alt text](https://github.com/user-attachments/assets/873563a2-1540-4dd2-955c-5d9dadae049f)
<br><br>
The setup function cannot be designated as async, so to use async/await syntax, another function must be defined and called within the useEffect callback. This could be done with an IIFE. Obviously defining the function outside useEffect would not solve this issue because the useEffect callback would still need to be designated as async.
<br><br>
![alt text](https://github.com/user-attachments/assets/6f9953bb-3998-4384-a678-84b9c3917c7c)
<br><br>
Error handling would also need to be added in with either a full try/catch, as shown above, or with chaining a catch statement onto the awaited function call.
<br><br>
![alt text](https://github.com/user-attachments/assets/f68fdd08-4de4-4b75-af24-7feb5b0fbf60)
<br><br>
By using then(), the extra wrapper doesn’t need to be created around the function call. There also isn’t a need to use the promise value again later in the setup function (setting state is the only action performed with this value) so this syntax fits the structure well.
### Code splitting
I specifically used Webpack and SWC to create static asset bundles. When it comes to code splitting, it’s straightforward to separate files like utilities and the data service layer into different bundles but splitting up a web of component dependencies is less straightforward. I decided to take advantage of dynamic imports to lazy load components that would likely not need to be immediately accessed by the user. This works as expected, with Webpack designating the dynamically imported file as a separate chunk, but if the application is truly modular with components spread out across multiple files this can create another problem with once again having too many smaller individual files. This remains the case even with minSize set lower on the SplitChunksPlugin object because the dynamic import is only importing one file at a time.
<br><br>
The way I addressed this was by creating a “lazy-index” barrel file in which the components intended to be lazy-loaded are imported, then exported. Then the main app.js file, which defines the router, can dynamically import this index file. This results in all of these lazy-loaded components being bundled together.
<br><br>
A typical barrel file.
<br><br>
lazy_index.js:
<br>
![alt text](https://github.com/user-attachments/assets/d8ed2da7-35bc-4d37-8e62-af6adb170a2a)
<br><br>
Any other exports, like loader or action functions, can be exported here as well (demonstrated in the third line).
<br><br>
app.js:
<br>
![alt text](https://github.com/user-attachments/assets/3b528a63-a9ad-4149-b11d-2d4c5a64ef30)
<br>
This structure avoids the typical barrel file pitfall of unnecessary importing by app.js because app.js would already be importing each of these component files anyway. The difference is that they are now recognized as one chunk by Webpack.
### &lt;ConfirmAction&gt; component
A critical UI feature of web forms is the ability to gracefully cancel filling out the form. Just as important as this feature, is a safeguard against accidentally canceling. I created a &lt;ConfirmAction&gt; component to address this, which defines a popup menu with a prompt and two buttons: one to cancel the confirmation and resume filling out the form and the other to cancel form submission.
<br><br>
There are two main structural aspects of the component. One is the process for confirming the action, and the other is how the component closes in response to events.
<br><br>
The base structure is as follows.
<br><br>
confirm_action.js:
<br>
![alt text](https://github.com/user-attachments/assets/d029df30-b672-4343-a77e-3972a9b59982)
<br><br>
The component takes 5 props:
- componentName: string<br>
&nbsp;&nbsp;&nbsp;&nbsp;The name of the parent component. See [modular CSS](#modular-css)
- buttonFor: string<br>
&nbsp;&nbsp;&nbsp;&nbsp;Essentially a name that can be used to further distinguish individual buttons in the event there are multiple buttons in a single component.
- buttonText: string<br>
&nbsp;&nbsp;&nbsp;&nbsp;The innerText of the button.
- confirmText: string<br>
&nbsp;&nbsp;&nbsp;&nbsp;Text that will appear in the popup after clicking the button. A prompt to further explain what will happen if the confirmation is completed.
- confirmHandler: (navigate: instanceOfUseNavigate) =&gt; void;<br>
&nbsp;&nbsp;&nbsp;&nbsp;A function that will be registered to the click event of the “Yes” button (i.e. the button that will complete the confirmation). It takes one argument, which is a function that will be used to navigate to a different page after confirmation. In this case I’m using an instance of React Router useNavigate() hook. The handler does not return anything.

Structuring the handler this way separates the navigation logic from the confirmation logic. The confirmHandler doesn’t need to know anything about how navigation works so if that logic needs to be changed it would only need to be updated in the &lt;ConfirmAction&gt; component and not every other component that instantiates the &lt;ConfirmAction&gt; component. Likewise, the &lt;ConfirmAction&gt; component doesn’t know anything about the confirmation logic, so that can be tailored to the parent component. The parent component only needs to define a confirmHandler with one parameter and it will receive a reference to the navigation function. In this example the userCache needs to be cleared out, but other parent components may perform other actions, or even perform no actions and just navigate away from the current form page.
Any potential issues with the HTTP request would be handled in the data service layer and then sent to the app error page so there’s no need to perform any checks on the res in then(). Passing in confirmHandler and navigate explicitly as arguments allow flexibility in case handleConfirmClick needs to be moved somewhere else.
<br><br>
The component can be called from its parent.
<br><br>
settings.js:
<br>
![alt text](https://github.com/user-attachments/assets/9ebc24d8-79cf-4c50-98b3-e234ad748682)
<br><br>
The rest of the <ConfirmAction> component logic defines its opening and closing behavior. There are a few parameters that need to be satisfied to make this work:
-	the form will not be submitted if the button is clicked to open the menu
-	when the menu is open, the menu buttons can be navigated to with the tab key
-	pressing the enter key while the outer button is focused will toggle the menu opening and closing
-	any time the menu is closed, associated event handlers need to be removed
-	pressing the escape key will close the menu
-	clicking outside the confirm-menu will close the menu
-	clicking inside the menu, but not on either the “Yes” or “No” button, will do nothing
-	clicking the “Yes” button will run the confirmHandler and close the menu
-	clicking the “No” button will close the menu
-	pressing shift + tab while the outer button is focused will close the menu and move focus back to the previous element
-	pressing shift + tab while the “Yes” button is focused will move focus back to the outer button and will keep the menu open
-	pressing the tab key while the “No” button is focused will close the menu and move focus to the next element
-	if the component is unmounted (a different page is navigated to) the event listeners will be removed gracefully and will not throw any errors

handleOuterButtonKeyDown and handleNoKeyDown are called to make sure the menu is closed when focus is moved to either the previous or next element respectively given normal keyboard navigation. The onClick handler for the “No” button just closes the menu.
<br><br>
confirm_action.js:
<br>
![alt text](https://github.com/user-attachments/assets/ee22754a-ed85-4887-9e68-7254fdf00629)
<br><br>
![alt text](https://github.com/user-attachments/assets/9ef52130-4a62-4a89-85ab-ab11896f0aa1)
<br><br>
Note: Adding type=’button’ removes the need to call Event.preventDefault() in click handlers registered to buttons inside the form. Technically this isn’t required for the “Yes” or “No” buttons because they are inside a positioned element. Event.preventDefault() is still needed in the click callback because this is registered to the document and if another button (like the submit button) is pressed the form will submit.
<br><br>
handleOuterButtonClick will add event listeners if it was previously closed and will remove event listeners if previously open. addListeners() and removeListeners() are the return values from a custom hook that I will describe more below.
<br><br>
confirm_action.js:
<br>
![alt text](https://github.com/user-attachments/assets/85c10137-5141-4e07-93be-1d19375ca154)
<br><br>
![alt text](https://github.com/user-attachments/assets/66c0d4f3-28e2-4107-8591-873b369ed632)
<br><br>
There are two global events that need to be considered. One is if a click happens outside of the menu, and the other is if the escape key is pressed.
<br><br>
A ref is used for both handlers:
<br>
![alt text](https://github.com/user-attachments/assets/59564d75-464e-42d4-a4b8-4d8c692a4d32)
<br><br>
![alt text](https://github.com/user-attachments/assets/11e590c1-b0e3-49bc-9eea-e5d6912f9ab3)
<br><br>
![alt text](https://github.com/user-attachments/assets/6067c9fe-3119-4f92-aacc-704e8e7accfb)
<br><br>
The logic in both handlers registered to these events is essentially the same. If confirmRef.current evaluates to falsy (meaning the menu is closed or not present on the page), remove each event listener. If the qualifying event happens, close the menu and remove each event listener. The first branch allows for graceful removing of the event handlers in the event the page is navigated away from. The second branch defines the desired behavior for the menu itself. Technically the separate if/else branches are not necessary in the callbacks because calling setConfirmVisible from another page (component) will not throw any errors, but this phrasing can make the intent more explicit: the first branch will run if the current page is navigated, the second will run on the current page. 
<br><br>
I created a hook for adding and removing event listeners to and from the document. The hook keeps track of registered event listeners with useRef and returns two functions that can be used to add and remove listeners.
<br><br>
manage_listeners.js:
<br>
![alt text](https://github.com/user-attachments/assets/83f368dc-f188-4f1d-9b2d-a8249d243fdd)
<br><br>
Note: Throwing a ReferenceError in removeListeners can help catch bugs in future development.
<br><br>
Back in the component, call the hook to access the functions.
<br><br>
confirm_action.js:
<br>
![alt text](https://github.com/user-attachments/assets/b6a6c6c3-6ca0-4a53-90b4-567dd7b0bc9c)
<br><br>
This setup is probably excessive for a component of this size and complexity, but if there were a lot of event listeners being used, this would end up being more scalable and extensible. This also makes removing event listeners more straightforward because there’s no need to remember whether useCapture has been set, which is needed by removeEventListener. For this example, the hook listener storage is only defined inside the hook itself, which means that any time it’s called it will create a new storage object. The storage can easily be made global by structuring it as a context. With that structure, it would be possible to remove an event listener in a component different from the one it was added in.
### Modular CSS
As a project like this grows, keeping CSS organized and clear of namespace collisions becomes critical. I mainly use classNames for styling because they have a lower specificity than IDs, which makes it easier to override styles in the event a style needs to have a higher importance. Each component I’ve written takes in a componentName prop that is used to construct modular classNames. The componentName would be a string that identifies the component’s parent component. Constructing classNames like this would also support further React extensibility especially if future logic requires distinguishing between parent components.
<br><br>
child_component.js:
<br>
![alt text](https://github.com/user-attachments/assets/f3ef8ea2-426b-4724-83ba-75427333d353)
<br><br>
register.js:
<br>
![alt text](https://github.com/user-attachments/assets/39595991-6194-4c10-9bfd-2bcf6640f46c)
<br><br>
Rulesets can then be constructed with any kind of modularizing syntax:
<br>
![alt text](https://github.com/user-attachments/assets/020e1a41-9c20-447c-8775-5a2981b98dfe)
<br><br>
While developing, I have separate CSS files for each component, but when they are bundled, namespace collisions become more likely. The only potential issue is if there are two components with the same name within a project.



