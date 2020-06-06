---
layout: post
title: Working with nested forms and a many-to-many association in Rails 4 📥
date: '2014-04-04 00:37:00'
tags:
- rails
---

Recently a project I was working on needed a _many-to-many_ relationship that would also store some extra data in the pivot table.

Rails provides helpers to make working with this sort of relationship a breeze but when you start to include the nested forms and requirement to add data to that connecting table the solution may not be that obvious.

I’ll be using Rails 4, the code will be the same for Rails 3.2 for the most part the major difference is [Strong Parameters](https://github.com/rails/strong_parameters) is now used in place of `attr_accessible`. You can find out [how to install Rails 4 yourself here](https://www.createdbypete.com/articles/ruby-on-rails-development-setup-for-mac-osx/).

## Getting started

For this example I’m going to use a Survey application, unfortunately this was a survey done in the street on paper and now the results need to be manually added to the system.

Each Survey will have some Questions, these Questions will be answered by a Participant.

So in this example we need an Answers table to be our many-to-many table that will link our Participant to our Question and keep the Answer the participant provided in an additional column.

So let’s start a new Rails application.

    rails new SurveyApp

Generate some models and scaffolds to save a little bit of typing later.


    rails generate scaffold Participant name
    rails generate scaffold Survey name
    rails generate model Question content:text survey:references
    rails generate model Answer question:references participant:references content:text

    rake db:migrate

First, we’ll sort out the models, the file names are above each class as a comment.

```ruby
# app/models/participant.rb
class Participant < ActiveRecord::Base
  has_many :answers
  has_many :questions, through: :answers
end

# app/models/survey.rb
class Survey < ActiveRecord::Base
  has_many :questions

  accepts_nested_attributes_for :questions
end

# app/models/question.rb
class Question < ActiveRecord::Base
  belongs_to :survey
  has_many :answers
  has_many :participants, through: :answers

  accepts_nested_attributes_for :answers
end

class Answer < ActiveRecord::Base
  belongs_to :participant
  belongs_to :question
end
```

You’ll notice I’m not worrying about validation in this guide because it’s a simple enough example and this post is concentrating on the nested forms and many-to-many associations.

You should be familiar with what you see here, I’ve used `through:` as this is recommended in the documentation as we have extra fields we want to access on the pivot table.

Now let’s tackle the controllers, in fact we only need to tackle the Survey controller.

```ruby
# app/controllers/surveys_controller.rb
class SurveysController < ApplicationController
  before_action :set_survey, only: [:show, :edit, :update, :destroy, :answers]

  # ... ignoring content that hasn't changed from scaffold

  def answers
    @participants = Participant.all
    @questions = @survey.questions
  end

  private

  # ... ignoring content that hasn't changed from scaffold

  # Never trust parameters from the scary internet, only allow the white list through.
  def survey_params
    params.require(:survey).permit(:name,
      :questions_attributes => [:id, :content,
        :answers_attributes => [:id, :content, :participant_id]
      ])
  end
end
```

Because of [Strong Parameters](https://github.com/rails/strong_parameters) replacing `attr_accessible` in Rails 4 we tell the application which attributes to allow through to our model to avoid mass-assignment security issues. The way it works is similar but you need to specify _everything_ this includes the attributes within our nested models. (Don’t forget the `id` attribute!)

Next we setup a [member route](http://guides.rubyonrails.org/routing.html#adding-more-restful-actions) we can use to enter our answers and associate them with a survey.

```ruby
# config/routes.rb
SurveyApp::Application.routes.draw do
  resources :surveys do
    get 'answers', on: :member
  end
  resources :participants
end
```

The _behind the scenes_ work is done so lets sort out our views. Specifically the form so we can add the answers

```erb
# app/views/surveys/answers.html.erb
<h1><%= @survey.name %> Answers</h1>

<%= form_for(@survey) do |f| %>
  <% @participants.each do |participant| %>
  <h3><%= participant.name %></h3>
  <table>
    <thead>
      <tr>
        <td>Questions</td>
        <td>Answer</td>
      </tr>
    </thead>
    <tbody>
      <% @questions.each do |question| %>
      <tr>
        <td><%= question.content %></td>
        <td>
        <%= f.fields_for :questions, question do |q| %>
          <%= q.fields_for :answers, question.answers.find_or_initialize_by(participant: participant) do |a| %>
            <%= a.text_area :content %>
            <%= a.hidden_field :participant_id, participant.id %>
          <% end %>
        <% end %>
        </td>
      </tr>
      <% end %>
    </tbody>
  </table>
  <% end %>
  <div class="actions">
    <%= f.submit %>
  </div>
<% end %>
```

What we have done there is create a table for the Survey model in the usual, then nested within that `fields_for` Questions and within that `fields_for` Answers. This allows Rails to make use of the `accepts_nested_attributes_for` method we used in the models.

For the Answers `fields_for` we are using the `find_or_initialize_by` method so that our answer `text_area` will populate with data if it’s available and if there isn’t a record for that Participant and Question combination it initializes a model so the form builder has an object to map on to.

You’ll also notice a `hidden_field` where we set the `participant_id` for the record to ensure the answer gets associated to a participant (`fields_for` will automatically create a `hidden_field` for `question_id` as we use that model to build the answers object, view source on the page and you will see).

The way I have chosen to display this is perhaps not the most efficient but it demonstrates how you might tackle this scenario where you need to display all these options and still handle the data submission. If you have another solution to this please let me know on [Twitter @createdbypete](https://twitter.com/createdbypete).
