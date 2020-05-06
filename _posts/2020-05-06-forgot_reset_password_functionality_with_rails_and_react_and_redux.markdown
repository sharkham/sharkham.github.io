---
layout: post
title:      "Forgot/Reset Password functionality with Rails and React (and Redux)"
date:       2020-05-06 20:53:58 +0000
permalink:  forgot_reset_password_functionality_with_rails_and_react_and_redux
---


For my final project with Flatiron, I have been building an app that tracks novel writing progress for a novel contest. It's actually been built for a specific novel contest, and through building the app, I knew my goal was to, as soon as I had passed the project requirements, actually deploy it for use. 

The biggest thing that stuck out to me as necessary for this was forgot/reset password functionality. I could do an admin controller later, fix my styling later, et cetera, but if I had live users and they were forgetting their passwords, this would be a problem. 

Many hours later, I had done enough reesarch and trial and error to build a solution. 

Before I get into it, a bit of a disclaimer--if you're logging in users through a Rails API backend, this is really a Rails issue. There are a few good tutorials on how to do this functionality with only Rails already. I drew heavily from them for my solution (and will link them later!), but what I wasn't able to find was something incorporating React to do this. Honestly, there are _good reasons for this_, which I will get into later! But, if you are looking to use Rails only for the backend of password resets and React with Redux for the front end, read on! 


**In the Rails API:**

First, some routes: 
```
  post 'api/v1/forgot_password' => "api/v1/passwords#forgot"
  post 'api/v1/reset_password' => "api/v1/passwords#reset"
```

There are two routes here because first you want the user to be able to submit their email through a forgot password action that will send a code to their email to start the process, and then once they have the code, you want them to be able to use it to submit their new password. 

Next, the user model needs a couple new columns. Whether you set this up through adding them after the fact or putting them directly in the original migration, they should look like this in your schema: 

```
    t.string "password_reset_token"
    t.datetime "password_reset_sent_at"
```

Then you need a new controller:

```
class Api::V1::PasswordsController < ApplicationController
  def forgot
    user = User.find_by(email: params[:_json])
    if user
      render json: {
        alert: "If this user exists, we have sent you a password reset email."
      }
      user.send_password_reset
    else
      #this sends regardless of whether there's an email in database for security reasons
      render json: {
        alert: "If this user exists, we have sent you a password reset email."
      }
    end
  end

  def reset
    user = User.find_by(password_reset_token: params[:token], email: params[:email])
    if user.present? && user.password_token_valid?
      if user.reset_password(params[:password])
        render json: {
          alert: "Your password has been successfuly reset!"
        }
        session[:user_id] = user.id
      else
        render json: { error: user.errors.full_messages }, status: :unprocessable_entity
      end
    else
      render json: {error:  ['Link not valid or expired. Try generating a new link.']}, status: :not_found
    end
  end

end
```

For my app, I have the passwords controller namespaced under `Api::V1`, but this is just preference. As long as the namespace is the same in the route and the controller (and the controller is under the proper `Api` and then `V1` folders, if applicable), it'll work. 

A lot of this code is drawn from these tutorials ([one](https://www.sitepoint.com/handle-password-and-email-changes-in-your-rails-api/), [two](http://railscasts.com/episodes/274-remember-me-reset-password?view=asciicast)), so I won't get too deep into specifics, but I recommend reading them if you're deploying this to get a better understanding of exactly what's going on! 

Briefly, the important thing about the `forgot` action is that you're finding the user by the email param that the user has submitted through a form (we'll get there), and then sending an email regardless of whether the email is in the database for security reasons, _but_ letting the user know about this so they don't spend forever waiting for an email only to realize later, oh no that was the wrong email I put in. When testing this, I recommend having different alerts for each case so you know which is which, but for deployment, this is what worked for me. 

The reset method is searching for a user by their email _and_ the `password_reset_token` that firing the `forgot` action sets on their account. This is a deviation from the tutorials I used for this part, and I'll get into why later. If the user exists and their token is valid, then the password reset fires, and if that works, they're also logged in by setting the `session[:user_id]` to their id. If the token is expired, or doesn't exist, or there's no user by that email, an error is rendered. 

Of course, to make this work we need some methods on the user model!

```
class User < ApplicationRecord
  ...
  has_secure_password
  validates :password, length: { minimum: 5, wrong_length: "Password must be at least 5 characters." }, if: :password

...

  def send_password_reset
    self.password_reset_token = generate_base64_token
    self.password_reset_sent_at = Time.zone.now
    save!
    UserMailer.password_reset(self).deliver_now
  end

  def password_token_valid?
    (self.password_reset_sent_at + 1.hour) > Time.zone.now
  end

  def reset_password(password)
    self.password_reset_token = nil
    self.password = password
    save!
  end

  private

  def generate_base64_token
    test = SecureRandom.urlsafe_base64
  end

end
```

`send_password_reset` sets the user's `password_reset_token` attribute to a randomly generated token, sets the `password_reset_sent_at` to the current time, and then after saving these to the user, sends an email to the user that will include this token and further instructions. More on that soon! The `password_token_valid` method checks if the token has been sent within the hour--if it's been longer than an hour, the application won't accept it. This sort of thing is personal preference, I've seen it set to longer than an hour, but I went with a shorter time window for extra security because some of the React implementation is a bit lower security compared to some other ways of doing this. The `reset_password` method sets the token to `nil` so that once it's used once to reset the password it can't be reset again, and it changes the user's password to whatever they entered in the form. 

The password validation line is _important_--without this you won't be able to set the `password_reset_token` and `password_reset_sent_at`. For more information on why, I have a separate blog post on that [here](https://sharkham.github.io/changing_attributes_on_a_user_in_rails_without_having_to_re-enter_password). 

The next thing to get set up is the Mailer functionality. First we need to generate a mailer: 

```
rails g mailer user_mailer password_reset
```

This will create a `user_mailer.rb` file under mailers, and two views for the `password_reset` email. This code goes in `UserMailer`--it's the method you're calling in `send_password_reset`: 

```
class UserMailer < ApplicationMailer

  def password_reset(user)
    @user = user
    mail to: user.email, subject: "Password Reset"
  end
	
end
```

The two views generated with the terminal command are really just html and plain text versions of the same email, and  your code for both of them should be the same other than that one can use html tags. 

```
Hi <%= @user.name %>,

You are receiving this email because you have requested a password reset for your Novel Tracker account.

Please use this code to reset your password: <%= @user.password_reset_token %>

This code will expire one hour from password reset request.

To reset your password please enter your code in the form here: http://localhost:3001/reset_password

If you did not request your password to be reset please ignore this email and your password will stay as it is.

```

You can use ERB tags to put in the user's name (or username, if your app uses that instead), and, importantly, the token. 

This is where my code diverges a little. [This tutorial](http://railscasts.com/episodes/274-remember-me-reset-password?view=asciicast) shows how to create a reset password view, and, even though the example there is done in a Rails-only project, many single page applications are not completely single page, and do something similar to this too--a reset password view through the API, and the rest of the app through the front end. 

Because I'm stubborn, and because I didn't want to figure out how to style a page rendered through Rails the same way I'd styled my React frontend, I decided to try to figure out how to do this through React instead. This led to a few specific choices here: 

One: exposing the password token in the email instead of including it as part of a dynamically generated link for the user to follow. Some apps do have both options, but mine only has one, because I wanted it to happen through a static link in React. This is because React is a bit strange about links. Well, not strange, but because it uses Client-side routing instead of Server-side routing, basically all of the app's content loads on the initial GET request to the server, and all of the routing from then is moving around within pages that are already downloaded from the beginning. 

There are ways around this--this [stack overflow thread](https://stackoverflow.com/questions/27928372/react-router-urls-dont-work-when-refreshing-or-writing-manually) gets into some. The specifics of figuring that out is beyond the scope of this blog post, but for my app, I have configured things so that any links a user doesn't need to be logged in to access can be navigated to by typing in the URL manually, and everything else (that requires a check for a logged in user) can't be. If you're going to use the way of doing things I'm outlining in this blog post for your project, make sure this is possible in your app!

Two: including a link to the reset password page. As previously described, if you can get it to work for your React app, it will be cleaner to do it this way and a bit more secure to have it not linked to from your front end. 

Having a static link to a reset password page does, however, make things a little less secure. This is why I have configured mine to require both the correct token and the matching user email in order to reset a user's password. 

Alright! The next step is configuring your settings so the mailing itself will work. REMEMBER: WHEN YOU CHANGE THESE SETTINGS, RESTART YOUR SERVER AFTERWARDS! I am embarrassed to admit this took me a lot of time in testing to figure out, so there is a reminder here! 

In config/environments/development.rb:
```
  #added settings
  config.action_mailer.perform_deliveries = true
  config.action_mailer.raise_delivery_errors = true

  config.action_mailer.delivery_method = :smtp
  host = 'localhost:3000'
  config.action_mailer.default_url_options = { :host => 'localhost:3000', protocol: 'http' }

  # SMTP settings for gmail
  config.action_mailer.smtp_settings = {
    :address              => "smtp.gmail.com",
    :port                 => 587,
    :user_name            => ENV["GMAIL_ACCOUNT"],
    :password             => ENV["GMAIL_PASSWORD"],
    :authentication       => "plain",
    :enable_starttls_auto => true
  }
```

Most of these settings are from [this article](https://dev.to/morinoko/sending-emails-in-rails-with-action-mailer-and-gmail-35g4), and I would recommend reading it as well for more information on how they work and for troubleshooting! Some of the trickier things here: your app needs somewhere to send mail from. This article recommends setting up a dedicated gmail account for this, which has worked for me. I have kept the information about it in my `.env` file, which I have added to my `.gitignore` file so it won't be uploaded to GitHub when I update my project there. 

The other recommendation from the article that I appreciated was setting up two-factor authentication and then setting an app password for apps to use for the email account--the app password is what I'm calling here with my `GMAIL_PASSWORD` variable. When I have tested this, the gmail account I've sent to still puts these emails in the spam folder, but they do go through at least! 

Also check out the previously linked article for advice on settings for your `config/environments/production.rb` file. As of writing this post I am still in the process of deploying my backend, so cannot yet speak to what changes work for me there. 


**In the React front-end** 

For reference, I didn't really code everything in Rails first and then everything in React after--it happened around the same time and involved a lot of testing throughout. But, for the purposes of this post, I thought it would be easier to separate these concerns to show how it works. 

So, with that said, forgot/reset password in React! First, you need a `ForgotPassword` component to display the form for users to request the code to their email: 

```
import React, { Component } from 'react';
import { forgotPassword } from '../helpers/passwords';
import { Link, withRouter } from 'react-router-dom';

class ForgotPassword extends Component {

  state = {
    email: ""
  }

  handleChange = (event) => {
    const { name, value } = event.target
    this.setState({
      [name]: value
    })
  }

  handleSubmit = (event) => {
    event.preventDefault()
    forgotPassword(this.state.email)
    this.setState({
      email: ""
    })
    this.props.history.push('/')
  }

  render() {
    return (
        <p>Request password reset:</p>
        <form onSubmit={this.handleSubmit}>
          <input required id="forgotpasswordemail" onChange={this.handleChange} name="email" placeholder="email" type="email" value={this.state.email}/>
          <button >Submit</button>
        </form>
    );
  }
}

export default withRouter(ForgotPassword);
```

![ForgotPassword component](https://lh3.googleusercontent.com/VyOyXwFwnJivaGDtqAmLNG1NAM2DBi8KK7eirB1BgPLFAm5qyVbujr0udNtNk2yDyVfna3G1KMz4zYJiPaJUpnXUYekLbcMzJj8EjapRmsqPe8gFrJw2L0ulhhbdjDM-zs_1m_3fug=w2400)
> My ForgotPassword component! Please note this isn't exactly the code above--I removed styling with reactstrap to make the code in this post easier to read.

This is a basic class component with a controlled form, but on submit, two important things are happening: 
1. The user's email is submitted to the `forgotPassword` method being called from the `helpers/passwords.js` file
2. The user is being redirected back to the home page with `this.props.history.push()`, and this method is possible to use here because of the last line: `withRouter(ForgotPassword)`. 

In that helpers file: 

```
const baseURL = "http://localhost:3000/api/v1"

export const forgotPassword = (email) => {
  return fetch(`${baseURL}/forgot_password`, {
    credentials: "include",
    method: "POST",
    headers: {
      "Content-Type": "application/json"
    },
    body: JSON.stringify(email)
  })
  .then(res => res.json())
  .then(response => {
    alert(response.alert)
  })
  .catch(console.log)
}
```

This method sends a `POST` request with the user's email to our `/forgot_password` route, and when it receives a response, it displays an alert with that response. Going all the way back to our `passwords_controller` in the Rails section of this post, that alert is `"If this user exists, we have sent you a password reset email."`

The next step of getting this set up in React is the `ResetPassword` component to display the form for users to enter the code they have received by email and use it to reset their password: 

```
import React, { Component } from 'react';
import { resetPassword } from '../helpers/passwords';
import { connect } from 'react-redux';


class ResetPassword extends Component {

  state = {
    token: "",
    email: "",
    password: "",
    password_confirmation: ""
  }

  handleChange = (event) => {
    const { name, value } = event.target
    this.setState({
      [name]: value
    })
  }

  handleSubmit = (event) => {
    event.preventDefault()
    const { password, password_confirmation } = this.state;
    if (password !== password_confirmation) {
      alert("Passwords don't match");
      this.setState({
        password: "",
        password_confirmation: ""
      })
    } else {
      this.props.resetPassword(this.state)
      this.setState({
        token: "",
        email: "",
        password: "",
        password_confirmation: ""
      })
    }
  }

  render() {
    return (
        <p>Reset Password:</p>
        <form onSubmit={this.handleSubmit}>
          <label for="token">Token:</label>
          <input required id="token" onChange={this.handleChange} name="token" placeholder="token" type="token" value={this.state.token}/>
          <p>The code that was emailed to you. This is case-sensitive.</p>
          <label for="email">Email:</label>
          <input required id="email" onChange={this.handleChange} name="email" placeholder="email" type="email" value={this.state.email}/>
          <label for="password">New password:</label>
          <input required id="password" onChange={this.handleChange} name="password" placeholder="password" type="password" value={this.state.password}/>
          <p>Set your new password here.</p>
          <label for="password_confirmation">Confirm new password:</label>
          <input required id="password_confirmation" onChange={this.handleChange} name="password_confirmation" placeholder="password confirmation" type="password" value={this.state.password_confirmation}/>
          <button type="secondary">Reset Password</button>
        </form>
    );
  }
}

const mapDispatchToProps = dispatch => {
  return {
    resetPassword: (credentials) => dispatch(resetPassword(credentials))
  }
}

export default connect(null, mapDispatchToProps)(ResetPassword);
```

![ResetPassword component](https://lh3.googleusercontent.com/7AvSs2-9Q8Q-CzQbv5AyWOogZmS0te-afTy9xC-mAgya4zDfq-IaQLxXFTZHpXgO3SLETXpG1woKgRjfTt3XF4IfYrTL-DBe8wb-r83ykhtCckVlqxb2dyDj-GBpp5iJ6qamjW7EXA=w2400)
> My ResetPassword component! As with the previous screenshot, this also includes reactstrap styling not included in my code here for simplicity's sake.

A little more is going on here! First, in `handleSubmit`, an alert fires and the `password` and `password_confirmation` fields reset to blank values if they don't match, to make sure the user is really resetting their password to the right thing. Second, if everything is in order on the form, `resetPassword` fires. 

A bit of a disclaimer on this one: `resetPassword` isn't quite what I would consider a Redux action, and I honestly haven't figured out yet whether it's a good idea to put it in an `actions` folder, as is Redux convention, or not. I am dispatching it instead of just calling it though, and mapping it to props through `mapDispatchToProps` and the `connect` function, and this is because after it fires, I want it to fire my `getCurrentUser` action and log the user in, and that is a Redux action. 

Here's what that looks like! 

```
import { getCurrentUser } from '../actions/currentUser'

const baseURL = "http://localhost:3000/api/v1"

export const forgotPassword = (email) => {
...
}

export const resetPassword = (credentials) => {
  return dispatch => {
    return fetch(`${baseURL}/reset_password`, {
      credentials: "include",
      method: "POST",
      headers: {
        "Content-Type": "application/json"
      },
      body: JSON.stringify(credentials)
    })
    .then(res => res.json())
    .then(response => {
      if (!!response.error) {
        alert(response.error)
      } else {
        alert(response.alert)
        dispatch(getCurrentUser())
      }
    })
    .catch(console.log)
  }
}
```

This method sends the credentials submitted in the `ResetPassword` component form to the `/reset_password` path as a `POST` request and returns a response. If there's an error in the action in `passwords_controller`, that will be an error, and this will show as an alert on the front end. If things go well on the back end, it show a "your password was reset!" alert and then checks the session for a current user. 

Getting into that functionality is a little beyond the scope of this blog post too, but I will demonstrate this part of my sessions functionality briefly to put the previous code in context: 

routes.rb:
```
  get '/api/v1/get_current_user' => "api/v1/sessions#get_current_user"
```

application_controller.rb
```
...
  def current_user
    User.find_by(id: session[:user_id])
  end

  def logged_in?
    !!current_user
  end
	...
```

sessions_controller.rb:
```
  def get_current_user
    if logged_in?
      render json: current_user
    end
  end
```

actions/currentUser.js:
```
... 
export const getCurrentUser = () => {
  return dispatch => {
    return fetch(`${baseURL}/get_current_user`, {
      credentials: "include",
      method: "GET",
      headers: {
        "Content-Type": "application/json"
      }
    })
    .then(res => res.json())
    .then(user => {
      if (user.error) {
        alert(user.error)
      } else {
        dispatch(setCurrentUser(user))
      }
    })
    .catch(console.log)
  }
}
```

So, the `getCurrentUser` action dispatches a `GET` request to the `get_current_user` action in the `sessions_controller`, and if there is currently a user in session--as a user is set in session in the `reset` action in the `passwords_controller` in the code at the beginning of this post--then it returns the user object, and uses that to set a current user in the Redux store, which, for the purposes of my app, is logging them in. 

As a final note, there is no redirect in my `ResetPassword` component because my app has conditional rendering for that page--once a user is logged in, they will be redirected away from routes that logged in users don't need to see anyway.

Phew! I think that's it. If you've made it this far, thank you for sticking it out, and I hope this helps if you're trying to implement something similar! 
