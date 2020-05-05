---
layout: post
title:      "Changing attributes on a user in Rails without having to re-enter password"
date:       2020-05-05 22:33:07 +0000
permalink:  changing_attributes_on_a_user_in_rails_without_having_to_re-enter_password
---


This is something that first came up for me during my Rails project for Flatiron, and I really wish I had figured it out then, but it feels fitting to have finally solved this mystery in my last weeks in the program. 

The issue is: you've set up a project with Rails. You include a user model, and add validations for the user's password. Then you want to make something about the user editable. A bio attribute, or something like that. You get all of the edit functionality set up, test out submitting it, and, error! A password hasn't been entered! 

This makes sense, because you didn't enter a password, and you don't want the user to have to enter a password to make changes to their profile when they're already logged in either! But because of the way the password validation is set up, Rails requires a password to be there to save the user. 

To fix this, include this line in your user model: 

```
  validates :password, length: { minimum: 5, wrong_length: "Password must be at least 5 characters." }, if: :password
```

The first bit is specific to my project, though having a minimum password length is probably a good idea anyway, but the last part is what does the job here: `if: :password`. This tiny bit of code means that if a password isn't included in the changed user object being submitted, then Rails won't worry about validating it. 

Finally, if you're dealing with strong params, include this in whatever controller is dealing with updating the user: 

```
if user_params[:password].blank?
  user_params.delete(:password)
end
```

This should work for projects with Rails as an API (my current project) as well as projects only using Rails (the project I may go back and add this functionality to!). 

I hope this helps anyone else struggling with this issue as I was! 

