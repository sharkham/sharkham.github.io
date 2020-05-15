---
layout: post
title:      "Deploying a React (with Redux)/Rails app with Netlify/Heroku"
date:       2020-05-15 19:18:17 -0400
permalink:  deploying_a_react_with_redux_rails_app_with_netlify_heroku
---


My final Flatiron project was made for use by an online novel contest, and after the project deadline, I set my own deadline of when I told the contest's organizer I would have the live website up by. This led to a lot of quick research and troubleshooting how to get a React/Rails app hosted, and another two weeks after submitting my final project, I had it up! The experience of seeing something I made in use has been very gratifying, and this blog post is an overview of what worked for me, in case it helps someone else!

**Initial set-up**

To start off, my app is built with React.js, Redux, and Rails 6. I built the front-end in a `project-name-frontend` folder with `create-react-app` and the backend in a `project-name-backend` with `rails new project-name-api --api --database=postgresql`. The frontend and backend are both hooked up to separate github repos. I'm also working off a Mac. This blog post will assume you have a similar setup in order for it to work as a tutorial! 

To hook up your Rails backend to Heroku, it is important it use Postgres instead of SQLite3 (the default) as a database. Adding the `--database=postgresql` takes care of a lot of the set up within your Rails project if you include this when you're building it, but I found [this article](https://medium.com/@noordean/setting-up-postgresql-with-rails-application-357fe5e9c28) helpful too for getting Postgres installed on my machine. You can skip a few of the steps if you start your Rails project setting the database to Postgres, but the rest of it still applies. 

**Deploying the backend** 

Alright, so you've built your React/Rails project, you have a Postgres database in Rails, everything is working in development, you're ready to deploy! 

The first step is getting your backend up on Heroku. Start by making an account on Heroku, and then navigate to [this tutorial](https://devcenter.heroku.com/articles/getting-started-with-rails6). It will prompt you to [install the Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli#download-and-install), log in on the command line with `heroku login`, and then configure your `config/database.yml` file. 

What is on the Rails 6 tutorial works for this, but there is a lot of text there, so for simplicity's sake, this is what worked for me: 
```
default: &default
  adapter: postgresql
  encoding: unicode
  username: <%= ENV['POSTGRES_USER'] %>
  password: <%= ENV['POSTGRES_PASSWORD'] %>
  pool: 5
  timeout: 5000
  host: <%= ENV['POSTGRES_HOST'] %>
development:
  <<: *default
  database: <%= ENV['POSTGRES_DEVELOPMENT_DB'] %>
test:
  <<: *default
  database: <%= ENV['POSTGRES_TEST_DB'] %>
production:
  <<: *default
  database: <%= ENV['POSTGRES_DB'] %>
```

Now, this relies on some environmental variables. You should have these in an `.env` file and that `.env` file added to your `.gitignore` so it won't show up when you push to github. 

For example: 
```
POSTGRES_USER='username'
POSTGRES_PASSWORD='password'
POSTGRES_HOST='localhost'
POSTGRES_DEVELOPMENT_DB='app_name_development_db'
POSTGRES_TEST_DB='app_name_test_db'
POSTGRES_DB='app_name_db'
```

Next, to deploy your app on Heroku, make sure you're in your `project-name-backend` directory and type `heroku create`. It should say something like: 

```
Creating app... done, radiant-sierra-11874
https://radiant-sierra-11874.herokuapp.com/ | https://git.heroku.com/radiant-sierra-11874.git
```
([source](https://devcenter.heroku.com/articles/getting-started-with-rails6))

Following along with the Heroku tutorial, you can check that the remote was added to your project correctly by typing `git config --list | grep heroku`. If you see `fatal: not in a git directory` you're not in the right directory. 

Otherwise, type `git push heroku master` to deploy your code. This will give you a long block of text, including some warnings at the end you might want to debug. 

If everything goes well there, you can migrate your database, and seed it if applicable: 

```
heroku run rake db:migrate
heroku run rake db:seed
```

This is an abbreviated run through that pulls out the specific steps from Heroku's "[Getting Started with Rails 6](https://devcenter.heroku.com/articles/getting-started-with-rails6)" article that worked for me, but I highly recommend the full article for more detail here. 

If all of this is working, you can visit your [Heroku dashboard](https://dashboard.heroku.com/apps) to see your app. 

In the settings tab, you can change the app's name: 

![Screenshot of Heroku settings](https://lh3.googleusercontent.com/7EpbPtat0I_7pHlDTDZJDghQMaDtugIQsKaMdru7lMKea0dlDXeZGmuUPH6VQAe6xkouuaC8_9cjmPEb_XcvSptV61HtGRsyKr2_UKXP81FTzCDp32o1lDqhUzeRQC4S__xnmZx0vw=w2400)

And in the deploy tab, you can connect your app to your Github repo so pushing changes there will push changes to the live Heroku app as well: 

![Screenshot of Heroku deploy settings](https://lh3.googleusercontent.com/kZYGZrNNKFtc0f3QOdBoIEkYbQHy-Zya7IfiC25Y2tLoqXipOGHboG5BDXdWVJi_wyVkBpw1HdIuyYZtGqdkMIfn9ME-ncZxeLpYgRycKheLg_15N-fMlu3wGx3_Vq_qb9j7rQH1Wg=w2400)

**Deploying the frontend** 

We'll get back to the backend later, but the next step to getting this app hooked up is deploying your React app through [Netlify](https://www.netlify.com/). It's possible to do the frontend through Heroku too of course, but I like Netlify for the frontend because it loads right away when you navigate to it. In the free version of Heroku, your server sleeps when it hasn't been pinged for a while, so hosting your frontend on Netlify will show your user the front page of your site right away while Heroku gets the backend up and running in the background. (With that said, I recommend using `componentDidMount` on your React app's `App.js` component, or whatever else loads first, so that the Heroku server gets booted starting from when a user first gets to your site.)

To get started with Netlify, create an account then click to "New Site from Git". Clicking on "Github" from the list of options will let you search your Github repos for `project-name-frontend`. The settings on the next page are fine, you can go ahead and "Deploy site" from there. Netlify has a [blog post](https://www.netlify.com/blog/2016/09/29/a-step-by-step-guide-deploying-on-netlify/) with an overview of this process, featuring more screenshots, that I found helpful as well! 

Once your Netlify app is up and running, you can change its name in the "General" section of settings, and then navigate to the "Build and Deploy" tab. Make sure the site is set up for continuous deployment, the first section there, and then scroll down to environment. 

Set an environment variable with a key of something like this: `REACT_APP_BASE_API_URL`, and set the value to the URL of your new Heroku app. 

The thing that I found when deploying my app is: running it on my local server in development, it uses environment variables from my `.env` file. Running it in production from Heroku and Netlify, the frontend and backend apps don't have access to any of these variables, so they have to be set through the Heroku and Netlify dashboards. This is actually *great*, because it's an easy way of making sure your frontend fetches from `localhost:3000` (or whatever port your backend is on in) in development and from `project-name-backend.heroku.app` in production, but it takes some configuring. 

In `project-name-frontend`, go into all of your files that make fetch requests. Change your base URL for these fetch requests to this: 

```
const baseURL = process.env.REACT_APP_BASE_API_URL
```

In React apps, environment variables are accessed through `process.env`, and if you've made your app with `create-react-app`, all environment variables need to be prefaced by `REACT_APP_` to work properly. ([More information here](https://medium.com/@tacomanator/environments-with-create-react-app-7b645312c09d)!)

From here, make an `.env.development` file in your `project-name-frontend` directory, add it to your `.gitignore` file, and add this environment variable there: 

```
REACT_APP_BASE_API_URL='http://localhost:3000/'
```

This should enable your frontend to properly `fetch` from your backend, from the local server in development and your heroku app in production! 

But, there's a problem here--the backend doesn't know to accept requests from your Netlify frontend yet! We have to go back and do more configuration there. 

***A note about Netlify:***

Before I go any further, I want to briefly mention that while Netlify loads faster than Heroku when first navigating to the live site, Netlify is definitely slower than Heroku to update after you've run `git push` and pushed changes through to it. I ran into a lot of issues debugging in deployment just because Netlify hadn't loaded the (working!) update I'd made to my code yet. 

So if you are refreshing your Netlify frontend to see if something has worked, you might need to wait a few minutes for the update to take! 

**More backend configuration**

Assuming your app was working in development before this, you should have your `/config/initializers/cors.rb` file configured. The `cors` file is where we tell the backend what requests to accept, so this is what needs to be reconfigured to get the Heroku app to accept `fetch` requests from the Netlify app. 

```
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins ENV['FRONT_END_URL']

    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      credentials: true
  end
end
```

Setting the `cors` file to allow origins `ENV['FRONT_END_URL']` means it will allow requests from whatever that environment variable is set to in `.env` in development, and whatever that environment variable is set to on Heroku in production. 

Add this line to your `.env` file (assuming you've set your port to 3001 on the frontend like I did): 

```
FRONT_END_URL='http://localhost:3001'
```

On the Heroku dashboard, go into settings, down to Config Vars, and create a new `FRONT_END_URL` config variable and set it to your Netlify app URL. 

Remember, modifications to the `cors.rb` file mean you need to restart your Rails server on the backend, and also, the change may take a minute or two to take effect in your Heroku app file too. 

But, this is it! Both apps have been deployed, and should be working properly! 

**The Redux issue**

Or, so I thought until I proudly sent the link to my website to the novel contest's organizer only to hear my beautiful app was only showing a blank page. Some poking around on my end trying to pull up the app in different browsers revealed I had the same problem too: the app was only displaying properly in Chrome. 

Eventually I figured it out: Redux Devtools, which were amazingly helpful while putting together my app, somehow meant that the app wasn't visible for any browser that did not have the devtools installed. I'm sure there's a way of configuring this so the devtools are included in development and not in production, but facing a deadline, I just removed them and everything worked fine. 

My code for creating my Redux store went from this: 

```
const store = createStore(rootReducer, compose(applyMiddleware(thunk), window.__REDUX_DEVTOOLS_EXTENSION__ && window.__REDUX_DEVTOOLS_EXTENSION__()))
```

to this:

```
const store = createStore(rootReducer, applyMiddleware(thunk))
```

And everything worked! 

I hope this is helpful to anyone else looking to deploy React/Rails apps! 
