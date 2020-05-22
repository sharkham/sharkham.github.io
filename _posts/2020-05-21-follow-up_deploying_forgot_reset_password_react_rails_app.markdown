---
layout: post
title:      "Follow-up: deploying Forgot/Reset Password (React/Rails app)"
date:       2020-05-22 00:18:15 +0000
permalink:  follow-up_deploying_forgot_reset_password_react_rails_app
---


In my recent [post on forgot/reset password functionality](https://sharkham.github.io/forgot_reset_password_functionality_with_rails_and_react_and_redux), I explored setting up a forgot/reset password function with a Rails backend and React frontend, and in my [post on deploying this project with Netlify and Heroku](https://sharkham.github.io/deploying_a_react_with_redux_rails_app_with_netlify_heroku), I focused on what it took to get the app up and running in production, but there wasn't really room for getting into what that meant for my forgot password functionality. This is what I'll explore here. 

Assuming everything is set up so your forgot/reset password function works in development, and assuming everything else works in deployment, there are only a few changes to make to your app to get the forgot/reset password function working in production. Testing this was the last phase of making sure my Novel Tracker app worked in production--I needed to make sure future users would be able to reset any forgotten passwords successfully before I let the novel contest know the app was ready for use! 

**On the backend **

First, in the Rails API, there are some settings to be added to `config/environments/production.rb`: 

```
  config.action_mailer.perform_caching = false

config.action_mailer.perform_deliveries = true
  config.action_mailer.raise_delivery_errors = true


  config.action_mailer.delivery_method = :smtp
  host = ENV["PRODUCTION_URL"]
  config.action_mailer.default_url_options = { host: host }

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

These are all of my settings in this file that have to do with `config.action_mailer`. Most of these should be the same as what you have in `config/environments/development.rb` except for these two: 

```
  host = ENV["PRODUCTION_URL"] 
  config.action_mailer.default_url_options = { host: host }
```

The host is no longer `localhost:3000`, it is whatever your production URL is. 

(For more information on set up with Action Mailer and Gmail, see [this article](https://dev.to/morinoko/sending-emails-in-rails-with-action-mailer-and-gmail-35g4).)

**On Heroku** 

Once `production.rb` is set up, the only other thing that needs to be configured is the environment variables in Heroku, referred to there as config vars. Navigate to your app on Heroku, go to the settings tab, and add these variables in the config vars section: 

```
PRODUCTION_URL: https://your-app-api.herokuapp.com
GMAIL_ACCOUNT: your.app.gmail.account@gmail.com
GMAIL_PASSWORD: yourgmailapppassword
```

The `PRODUCTION_URL`, your app's URL on Heroku, is the `host` for Action Mailer, and your `GMAIL_ACCOUNT` and password are whatever email address you want your app to send the reset password emails from. The `GMAIL_ACCOUNT` and `GMAIL_PASSWORD` can be the same variables in development and production, but, as Heroku can't access your environment variables in the `.env` file listed in your `.gitignore`, you need to provide them to Heroku directly as config vars for them to work in production. 

Along these lines, we need to add one more config var to Heroku to get everything working: 

```
RESET_PASSWORD: https://your-app-frontend.netlify.app/reset_password
```

This is a link to the `reset_password` page/component in your Netlify app, and Heroku needs to know about it because it's what will be sent out to users in reset password emails. But, there's one final thing that needs to be fixed to get this link working. 

**On the frontend** 

React uses client-side routing, which means, basically, it makes one request to the server to grab everything when you first navigate to the app, and after that, when clicking a link made with `react-router-dom`, the URL changes locally, and so does the component being displayed, if you've set things up that way--no other request to the server is fired. 

There are multiple workarounds to this, many of them detailed [here](https://stackoverflow.com/questions/27928372/react-router-urls-dont-work-when-refreshing-or-writing-manually), but if you're using `create-react-app` and Netlify, there's an easy one that works for at least some components.

Create a `public/_redirects` file in the main directory of your front end app, and include this code in it:

```
/*  /index.html  200
```
([Source](https://create-react-app.dev/docs/deployment/#netlify-https-wwwnetlifycom))

This will enable users to navigate directly to your `/reset_password` link, and any other links (at least in my project) that don't use conditional rendering to only allow logged-in users to see them. 

**That's it!**

This is everything I did to get this part of the app working! There are other tutorials on all of this out there that I've found very useful, but this concludes my series of blog posts on this particular use case, to help myself remember what I did and in case anyone else finds these specific steps useful! (And this is the demo version of the app in question, for any readers who wants to see some of it working on the frontend: http://novel-tracker-app.netlify.app/)
