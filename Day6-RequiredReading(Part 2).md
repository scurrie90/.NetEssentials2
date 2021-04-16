# .NET Essentials 2 - Session 6 (Client)
## Learning Outcomes

*  [Configuring JWT on the Client](#1)
*  [Add Register and Login To Navbar](#2)
*  [Add Register Component](#3)
*  [Add Login Component](#4)
*  [Add Client Side Validation](#5)
*  [Update a Fetch to Pass the Bearer Token](#6)

---

<a name="1"></a>
## Configuring JWT on the Client
Once you have add JWT generation and authentication on the server, we must now incorporate these features in our client app. Similar to our test in Swagger and Postman...if the client request do not have the token from the server, then they will not be allowed to interact with the authenticated endpoint(s).

### Browser Caching with SessionObject
It is possible to store content in the browser cache with the JavaScript sessionStorage object. These will come in handy when we recieve the JWT from the Register/Login we can setItem, and when we want to make future api requests during our session we can reference the JWT using getItem...

1. The setItem() function stores a string to the cache. 
```javascript
sessionStorage.setItem("MY_KEY", JSON.stringify(json));
```
2. The getItem() function of sessionStorage reads the content:
```javascript
let cachedItem = sessionStorage.getItem("MY_KEY");
```
3. To convert the item back into a JavaScript object use JSON.parse():
```javascript
let cachedObj  = JSON.parse(cachedItem);
```

---

<a name="2"></a>
## 2) Add Register and Login To Navbar
By adding our AuthController we have enabled our clients to make Register and/or Login requests to our API, where they will be provided with the JWT token for future calls.
Add a register and login navigation and page to your React app:

`Navbar.js`
```javascript
<div className="navbar-end">
    <div className="navbar-item">
        <div className="auth-buttons">
            <a href="/register" className="button is-light is-success"><strong>Register</strong></a>
            <a href="/login" className="button is-light is-link">Log in</a>
        </div>
    </div>
</div>
```

Replace the default styles in `App.css` with the following:
```css
.navbar, .navbar-end, .navbar-menu, .navbar-start {
  align-items: stretch !important;
  display: flex !important;
}

.navbar-menu {
  flex-grow: 1 !important;
  flex-shrink: 0 !important;
  box-shadow: none !important;
  padding: 0 !important;
}

.navbar-item {
  display: flex !important;
}

.navbar-end {
  justify-content: flex-end !important;
  margin-left: auto !important;
}

.button {
  margin-left: 1em;
}
h1 {
  font-size: 2em;
}
```

![](https://i.imgur.com/33pkAHn.png)

---

<a name="3"></a>
## 3) Add Register Component
Add the Route for the Register component to your `App.js`
```javascript
import Register from './components/auth/Register';

...

<Route exact path="/register" component={Register} />
```

Add the component in a new auth folder within components (matching the previous import path):
```javascript
import React, { Component } from 'react';

class Register extends Component { 
  //state variables for form inputs and errors
    state = {
    email: "",
    password: "",
    confirmpassword: ""
  }

  handleSubmit = async event => {
    //Prevent page reload
    event.preventDefault();
    
    //Perform Validation here
    
    //Integrate Auth here on valid form submission
  };

  onInputChange = event => {
    this.setState({
      [event.target.id]: event.target.value
    });
  }

  render() {
    return (
      <section className="section auth">
        <div className="container">
          <h1>Register</h1>
          <form onSubmit={this.handleSubmit}>
            <div className="field">
              <p className="control">
                <input 
                  className="input" 
                  type="email"
                  id="email"
                  placeholder="Enter email"
                  value={this.state.email}
                  onChange={this.onInputChange}
                />
              </p>
            </div>
            <div className="field">
              <p className="control">
                <input 
                  className="input" 
                  type="password"
                  id="password"
                  placeholder="Password"
                  value={this.state.password}
                  onChange={this.onInputChange}
                />
              </p>
            </div>
            <div className="field">
              <p className="control">
                <input 
                  className="input" 
                  type="password"
                  id="confirmpassword"
                  placeholder="Confirm password"
                  value={this.state.confirmpassword}
                  onChange={this.onInputChange}
                />
              </p>
            </div>
            <div className="field">
              <p className="control">
                <button className="button is-success">Register</button>
              </p>
            </div>
          </form>
        </div>
      </section>
    );
  }
}
export default Register;
```

![](https://i.imgur.com/bHyVjgs.png)

### Add the Register Api Call
Within the handle submit, where the comment for Integrate Auth here... is, send the POST request to the api's Register route:
```javascript
//Integrate Auth here on valid form submission
fetch('https://netcoreapi1.azurewebsites.net/Auth/Register', {
    method: 'POST',
    headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json',
    },
    body: JSON.stringify({
        Email: this.state.email,
        Password: this.state.password,
        ConfirmPassword: this.state.confirmpassword
    })
})
// Response received.
.then(response => response.json())
// Data retrieved.
.then(json => {
    console.log(JSON.stringify(json));
    // Store token with session data.
    if(json["status"]=="OK") {
        sessionStorage.setItem('bearer-token', json["token"]);
        console.log(sessionStorage.getItem('bearer-token'))
    }
    else {
        // error message handling
        console.log('Error in Auth/Register');
    }
})
// Data not retrieved.
.catch(function (error) {
    console.log(error);
})
```

### Test Sending a Registration
Send a registration request for a new user, the response should provide you with a JWT.

![](https://i.imgur.com/exsio8b.png)

---

<a name="4"></a>
## 4) Add Login Component
Add the Route for the Login component to your `App.js`
```javascript
import Login from './components/auth/Login';

...

<Route exact path="/login" component={Login} />
```

Add the component in a new auth folder within components (matching the previous import path):
```javascript
import React, { Component } from 'react';

class Login extends Component {
  state = {
    email: "",
    password: ""
  };

  handleSubmit = async event => {
    //Prevent page reload
    event.preventDefault();

    //Form validation
    
    //Integrate Auth here on valid form submission
    
  };

  onInputChange = event => {
    this.setState({
      [event.target.id]: event.target.value
    });
  };

  render() {
    return (
      <section className="section auth">
        <div className="container">
          <h1>Log in</h1>
          <form onSubmit={this.handleSubmit}>
            <div className="field">
              <p className="control">
                <input 
                  className="input" 
                  type="text"
                  id="email"
                  placeholder="Enter email"
                  value={this.state.email}
                  onChange={this.onInputChange}
                />
              </p>
            </div>
            <div className="field">
              <p className="control">
                <input 
                  className="input" 
                  type="password"
                  id="password"
                  placeholder="Password"
                  value={this.state.password}
                  onChange={this.onInputChange}
                />
              </p>
            </div>
            <div className="field">
              <p className="control">
                <button className="button is-success">
                  Login
                </button>
              </p>
            </div>
          </form>
        </div>
      </section>
    )
  }
}

export default Login;
```

![](https://i.imgur.com/Ity5ZfT.png)


### Add the Login Api Call
Within the handle submit, where the comment for Integrate Auth here... is, send the POST request to the api's Login route:
```javascript
//Integrate Auth here on valid form submission
fetch('https://netcoreapi1.azurewebsites.net/Auth/Login', {
    method: 'POST',
    headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json',
    },
    body: JSON.stringify({
        Email: this.state.email,
        Password: this.state.password
    })
})
// Response received.
.then(response => response.json())
// Data retrieved.
.then(json => {
    console.log(JSON.stringify(json));
    // Store token with session data.
    if(json["status"]=="OK") {
        sessionStorage.setItem('bearer-token', json["token"]);
        console.log(sessionStorage.getItem('bearer-token'))
    }
    else {
        // error message handling
        console.log('Error in Auth/Login');
    }
})
// Data not retrieved.
.catch(function (error) {
    console.log(error);
})
```

### Test Sending a Login
Send a login request for an existing user, the response should provide you with a JWT.

![](https://i.imgur.com/iyPDKWM.png)

---

<a name="5"></a>
## 5) Add Client Side Validation
Before we move away from our Auth forms, we should provide some client-side validation for both forms.

![](https://i.imgur.com/JOklZNF.png)

Inside the components folder create a new utility folder called util. Inside util we will add a utility method called validateForm:

`src/components/util/Validation.js`
```javascript
// currently checking for blank fields and password match
function validateForm(event, state) {
    // clear all error messages
    const inputs = document.getElementsByClassName("is-danger");
    for (let i = 0; i < inputs.length; i++) {
      if (!inputs[i].classList.contains("error")) {
        inputs[i].classList.remove("is-danger");
      }
    }
    // validate email is provided if in state (Register/Login)
    if (state.hasOwnProperty("email") && state.email === "") {
      document.getElementById("email").classList.add("is-danger");
      return { blankfield: true };
    }
    // validate password is provided if in state (Register/Login)
    if (state.hasOwnProperty("password") && state.password === "") {
      document.getElementById("password").classList.add("is-danger");
      return { blankfield: true };
    }
    // validate confirmpassword is provided if in state (Register)
    if (state.hasOwnProperty("confirmpassword") && state.confirmpassword === "") {
      document.getElementById("confirmpassword").classList.add("is-danger");
      return { blankfield: true };
    }
    // validate confirmpassword matches password if BOTH are in state (Register)
    if (
      state.hasOwnProperty("password") &&
      state.hasOwnProperty("confirmpassword") &&
      state.password !== state.confirmpassword
    ) {
      document.getElementById("password").classList.add("is-danger");
      document.getElementById("confirmpassword").classList.add("is-danger");
      return { matchedpassword: true, blankfield: false };
    }
    return;
  }

  export default validateForm;
```

We can pass the values as properties from a form component to render a FormErrors component as a section on any form page:
```javascript
import React from "react";

function FormErrors(props) {
  // currently only validating for two types of errors, blankfield and mismatched password on Register
  if (
    props.formerrors &&
    (props.formerrors.blankfield || props.formerrors.matchedpassword)
  ) {
    return (
      <div className="error container help is-danger">
        <div className="row justify-content-center">
          {props.formerrors.matchedpassword
            ? "Password value does not match confirm password value"
            : ""}
        </div>
        <div className="row justify-content-center help is-danger">
          {props.formerrors.blankfield ? "All fields are required" : ""}
        </div>
      </div>
    );
  } else {
    return <div />;
  }
}

export default FormErrors;
```

### Add FormErrors Into Each Relevant Component
```javascript
import FormErrors from "../FormErrors";

...

//state variables
errors: {
    blankfield: false,
    matchedpassword: false
}

...

//JSX
<FormErrors formerrors={this.state.errors} />
```

### Fire the Validation Utility Function
```javascript
import Validate from "../util/Validation";

...

//in the handleSubmit just before the fetch...
const error = Validate(event, this.state);
if (error) {
    this.setState({
        errors: { ...this.state.errors, ...error }
    });
} else {
// do the fetch ...
```

![](https://i.imgur.com/7Xap0iA.png)

### Clear Errors for User
```javascript
// helper function...be sure to list the state variables specific to the form
clearErrors = () => {
    this.setState({
        errors: {
            blankfield: false
            //matchedpassword: false
        }
    });
};

...

//in the handleSubmit before you start validation
this.clearErrors();

...

//add to the end of the onInputChange. As the user fixes the error(s) they are cleared
document.getElementById(event.target.id).classList.remove("is-danger");
```

---

<a name="6"></a>
## 6) Update a Fetch to Pass the Bearer Token
Currently we have only set up Authorization for our GetAll end point, however now that all the heavy lifting is done (generating and recieving the JWT), we could easily add this to any endpoint (or entire controller) within the api.
Update the fetch call inside the fetchTodos function to include the required Authorization header:

```javascript
fetch(BASE_URL+'todo', {
    method: 'GET',
    headers: {
        'Authorization': `Bearer ${sessionStorage.getItem('bearer-token')}`
    }
})
...
```
