---
layout: post
title:      "Creating two resources 'at once' in Rails"
date:       2019-12-13 21:00:24 -0500
permalink:  creating_two_resources_at_once_in_rails
---


For my Rails project for Flatiron, the thing that stumped me for the longest time was my models. One of the requirements was using a `has-many-through` relationship, and not only that, the join table had to have a user-submitted attribute on it. It couldn't be purely a join table, only used in the back end to hook up two models and not showing up in the actual front end of the app in any way. 

As what I wanted to make was a collaborative writing app, for writers of group novels to be able to easily work on their projects with their collaborators online, I needed users to have many novels and novels to have many users as well. What I eventually settled on was a join table of membership--a user could have an "admin" membership to a novel, with full access to editing and deleting functions, or a "member" membership to a novel, able to view and edit pieces of it but with less permissions overall. 

All of this seemed fine and good until I realized that this meant a couple of things that would make my project more complicated. 

1. In order for a user to "have" a novel, the membership associating the user and the novel needed to be created at the same time as the novel was. 
2. In order for a user to be able to edit the novel after, according to the app I was designing, this membership not only needed to be created, but give the user an admin role at the time of creation. 
3. I couldn't actually create the novel and the membership at the *same* time, because in order for a membership, which is really just a fancy join table with a role attached to it, to be created, it would need a novel id. Which required the novel be created and saved first. 
4. Oh no. 

Luckily, with some help from my instructor, classmates, and a lot of googling, I eventually figured this out! 

The first step was making sure my novel model would accept nested attributes for memberships, because even if the novel and the membership couldn't be created at exactly the same time, it would need to be in the same action, and therefore in the same form: 

```
# in novel.rb model: 
  accepts_nested_attributes_for :memberships

```

The next step was my novel#new form (at a nested route of: `/users/:id/novels/new`): 

```
<%= form_for(@novel, url: user_novels_path(@user)) do |f| %>
  <%= f.fields_for :membership do |mem| %>
    <%= mem.hidden_field :user_id, :value =>@user.id %>
    <%= mem.hidden_field :role, :value => "Admin" %>
  <% end %>

  <%= f.label :title %><br>
  <%= f.text_field :title %><br>
  <br>
  <%= f.label :summary %><br>
  <%= f.text_area :summary %><br>
  <br>
  <%= f.submit %>

<% end %>
```

This got a little tricky. At first I tried doing fields for `@membership`, and setting this as `Membership.new` in the novels controller, but I ran into some issues here. Luckily, `fields_for` is smart and does a lot of things! Getting into all of them is beyond the scope of this blog post, but one of the them is that it will accept a symbol name for a model instead of an instance of it. This allows us all of the functionality of making fields to accept attributes for the new membership object--without the object having to be created first. That comes next! 

The other important thing this form is doing is passing the id of the user making this novel object into the membership params so it can be used to create the membership in the controller. The novel doesn't have an id yet, but the user does, and we need both of these. It's also passing in the "Admin" role in by a hidden field as well. (Because of the way the app is set up, any user creating a novel is its admin, otherwise they wouldn't be able to edit or delete the novel--I didn't want any choices there.)

Finally, the controller methods, in `novels_controller.rb` in my app: 

```
  def new
    @novel = Novel.new
		@user = User.find_by(id: params[:user_id])
  end

  def create
    @novel = Novel.new(title: novel_params[:title], summary: novel_params[:summary])
    if @novel.save
      @novel.memberships.build(role: novel_params[:membership][:role], user_id: novel_params[:membership][:user_id])
      @novel.save
      redirect_to novel_path(@novel)
    else
      flash[:errors] = @novel.errors.full_messages
      render :new
    end
  end
	
	private

  def novel_params
    params.require(:novel).permit(
      :title,
      :summary,
      membership: [
        :id,
        :role,
        :user_id
      ] )
  end
```

The new method only needs to create a new novel instance for `form_for`, as `fields_for` is content with the symbol for membership because of the association. 

In the create method, first a novel is created from the strong params passed to the controller. Because there were params for the novel and also the membership getting passed in, this got a little complicated. The novel didn't need the membership params to be created; the membership needed those and it had to be created separately! How I ended up getting this to work was selecting the specific params for creating a novel from the `novel_params` method.

If, and only if, the novel saved, I could have the controller build a membership off that novel. Using the build method ensured that the membership would be created with an association already set up--the `novel_id` is present in this case as membership is being built off of the novel. After that, the membership's other attributes are pulled from the params passed in through `novel_params`: its role and the other id necessary for this object to save: the `user_id`. 

After this, the novel needs to be saved again--`build` as a method creates the associated object but doesn't save it--and then we're in business! A novel created with an id, and a membership created joining that novel to the user creating it, with an Admin role and privileges for the user in question. 

Note: Because in this app a user can only make a novel they own, `current_user.id` could have been used in the controller method instead of relying on getting the user's id from the params on the form. However, I have shown it this way here because this is how I initially got it to work, and because it might be useful for someone else trying to create two objects of a different sort 'at the same time', where the 'user' id is not a guarantee but where the objects need to be linked like my objects are here. 

If a novel could be created in association with any user, as it could earlier in the app's development, passing the `user_id` in as a param was the only way to get at which user was supposed to be associated with it from the `users/:id/novel/new` nested route. As HTML is a stateless protocol, the novel, once the form was submitted, wouldn't otherwise be able to access this information.

Learning Rails has been a journey so far, and there's still a lot I look forward to learning about this framework. It's very likely there's a cleaner way out there to achieve creating multiple objects at the 'same time' in a `has_many_through` association as I've done here, but that's part of the beauty of Ruby--there's always more than one way to do something. In the meantime, I hope this is helpful to anyone else who's struggling with this issue as I struggled with it! 
