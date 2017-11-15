# Step 02 - Authentication

AWS Amplify solved the authentication for developers. Let's use it.

* [1. Prepare](#1-prepare)
* [2. Configure AWS Amplify](#2-configure-aws-amplify)
* [3. Add Authenticator](#3-add-authenticator)
* [4. Greetings](#4-greetings)
* [5. Replace Sign In](#5-replace-sign-in)
* [6. Turn LoginForm into AuthPiece](#6-turn-loginform-into-authpiece)
* [7. Home page aware of authState](#7-home-page-aware-of-authstate)
* [8. Run app](#run-app)

## 1. Prepare

Install package, core library and react specific.
```
npm install --save aws-amplify
npm install --save aws-amplify-react
```

**Create a AWS Mobile Hub project**

Go to [Mobile Hub Import](../mobile-hub-import) to quickly setup the backend on AWS.

**Download config file**

Once project created, open "Hosting and Streaming"
![Hosting and Streaming](host_and_streaming.png)

At the bottom of page, "Download aws-exports.js file".
![Download aws-exports.js file](download_aws_exports.png)

Download and save to `src` folder

## 2. Configure AWS Amplify

Open `src/App.js`, add these lines
```
import Amplify from 'aws-amplify';
import aws_exports from './aws-exports';

Amplify.configure(aws_exports);
```

## 3. Add Authenticator

Open `src/modules/Login.jsx`, change content to:
```
import React, { Component } from 'react';

import { Authenticator } from 'aws-amplify-react';

export default class Login extends Component {
    render() {
        return <Authenticator />
    }
}
```

Now `npm start`. Login becomes real

<image src="authenticator.png" width="400px" />

Got ahead sign up and sign in. Create a test user.

## 4. Greetings

Notice after sign in, there is a sign out button. It makes sense to have sign out button. However in our case the place is not right.

**Hide Greetings**

Let's hide this one. `src/modules/Login.jsx` becomes:
```
import React, { Component } from 'react';

import { Authenticator, Greetings } from 'aws-amplify-react';

export default class Login extends Component {
    render() {
        return <Authenticator hide={[Greetings]} />
    }
}
```

`Authenticator` is composed of a group of pieces, `Greetings` is one of them. `hide` defines a list of pieces to be hidden. Try put `SignIn` in list.

**Greetings on Menu**

What we actually want. Is the greetings on Menu, top-right corner. So we want to edit where the Menu is, `App.js`.

First import Greetings
```
import { Greetings } from 'aws-amplify-react';
```

The default styling doesn't fit in our UI, lets add the menu item and remove default theme of Greetings.
```
const GreetingsTheme = {
    navBar: {
    },
    navRight: {
    },
    navButton: {
        border: '0',
        background: 'white',
        color: 'blue',
        borderBottom: '1px solid',
        fontSize: '0.8em'
    }
}

...

    <Menu.Menu position="right">
        <Menu.Item>
            <Greetings theme={GreetingsTheme} />
        </Menu.Item>
    </Menu.Menu>
```

**Custom Greetings**

Change the greetings
```
    <Greetings
        theme={GreetingsTheme}
        outGreeting="Welcome"
        inGreeting={(username) => 'Hi ' + username}
    />
```

## 5. Replace Sign In

The default Sign In form is good. However it doesn't fit with our theme. Let's add a Semantic UI [LoginForm](https://react.semantic-ui.com/layouts/login).

Save the form to `src/components/LoginForm.js`.

Then modify `src/modules/Login.jsx`, hide the default SignIn, add our LoginForm
```
import { Authenticator, Greetings, SignIn } from 'aws-amplify-react';

    render() {
        return (
            <Authenticator hide={[Greetings, SignIn]}>
                <LoginForm />
            </Authenticator>
        )
    }
```
Now, looks better

<image src="login_form.png" width="400px" />

## 6. Turn LoginForm into AuthPiece

The LoginForm actually doesn't do anything at this moment. We need to hook it up with Amplify Auth.

First we turn LoginFrom from const to component. `AuthPiece` is the base class in Amplify Auth components which has some built-in functions to help integrate with Auth. Let's extend from it.
```
import { Auth, Logger } from 'aws-amplify';
import { AuthPiece } from 'aws-amplify-react';

const logger = new Logger('LoginForm');

class LoginForm extends AuthPiece {
    constructor(props) {
        super(props);

        this.signIn = this.signIn.bind(this);
    }

    signIn() {
        const { username, password } = this.inputs;
        logger.debug('username: ' + username);
        Auth.signIn(username, password)
            .then(data => this.changeState('signedIn', data))
            .catch(err => logger.error(err));
    }

    ...

    render() {
        ...
              <Form.Input
                fluid
                icon='user'
                iconPosition='left'
                placeholder='Username'
                name="username"
                onChange={this.handleInputChange}
              />
              <Form.Input
                fluid
                icon='lock'
                iconPosition='left'
                placeholder='Password'
                type='password'
                name="password"
                onChange={this.handleInputChange}
              />
              <Button
                    color='teal'
                    fluid
                    size='large'
                    onClick={this.signIn}
              >Login</Button>
        ...
    }
```

Explain a little bit.

1. `handleInputChange` is from `AuthPiece`, it saves input value to `this.inputs` with its name.
2. On button click, `signIn` got called, it reads input values from `this.inputs` then call `Auth.signIn`
3. Logger is an organized way of logging. Type `LOG_LEVEL = 'DEBUG'` in console log to see debug logs.

Now run app, login. From Greetings on top-right corner we can see LoginForm works. But why is it still show up after sign in success.

**Hide LoginForm after Sign In**

Every AuthPiece got `authState` property. So just check `authState` in `render` method
```
    render() {
        const { authState } = this.props;
        if (authState === 'signedIn') { return null; }

        ...
```

## 7. Home page aware of authState

Now sign in works. How does Home page know if user signed in or not?

Remember we have a `Greetings` on menu? Listen to its `onStateChange` event, then pass to Home component in router.

In `src/App.js`
```
    constructor(props) {
        ...
        this.handleStateChange = this.handleStateChange.bind(this);
    }

    handleStateChange(authState, authData) {
        this.setState({
            authState: authState,
            authData: authData
        });
    }

    
```

```
        <Greetings
            theme={GreetingsTheme}
            outGreeting="Welcome"
            inGreeting={(username) => 'Hi ' + username}
            onStateChange={this.handleStateChange}
        />
```

```
        <Route exact path="/" name="home" render={(props) => (
            <Home {...props} authState={authState} authData={authData} />
        )}/>
```

Then, in `src/modules/Home.jsx`, just check `authState` property
```
    render() {
        const { authState, authData } = this.props;
        return (
            <div id="home-module">
                <Header as="h1">Home</Header>
                <div>{authState}</div>
            </div>
        );
    }
```

## 8. Run app
```
npm start
```