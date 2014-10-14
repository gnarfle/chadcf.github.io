---
layout: post
title: JqueryUI datepicker with rails and simple_form - the easy way
created: 1334698751
---
If you have a rails app, you've no doubt noticed that the default select date pickers are a little less than ideal. You may have just thrown in jQueryUI and replaced them with a text input type, and perhaps used a date format that rails understands because the more common (in the US) option of mm/dd/yyyy is parsed as dd/mm/yyyy. 

Well here's an easy fix. This is based on simple_form, if you are using formtastic or even stock form helpers it's easy enough to do the same thing. We're going to create a simple datepicker input type to make it very easy to create a nice date input in a form, and use a little magic and jQuery's altField options so that we can display mm/dd/yyyy to the user and send yyyy-mm-dd to rails.

First, the input type:

<b>app/inputs/datepicker_input.rb</b>
<code>
class DatepickerInput < SimpleForm::Inputs::Base
  def input
    @builder.text_field(attribute_name, input_html_options) + \
    @builder.hidden_field(attribute_name, { :class => attribute_name.to_s + "-alt"}) 
  end
end
</code>

Very straightforward. All we do is define a datepicker input type (which simple_form will automatically give a datepicker class to) which outputs a normal text_field. Next, to this we also output a hidden field with the same name. Note that this will have the same id as the text field which isn't really good, but you can pass in an :id attribute if it bothers you. This allows us to have a hidden input for jQuery to stash a rails compatible value in, while having a text field with the widget to present a nice selector to the user. 

To use this is quite simple, just use as you would any other field:

<code>
  = f.input :date, :as => :datepicker
</code>

Done!

Next we need a bit of javascript to wire this up. Since we want a general purpose solution here, we don't want to implement specific id selectors to initialize our inputs. Instead, we will simply find all of our input.datepicker elements, loop through them and wire them up with an altField of the next DOM sibling.

<b>app/assets/javascripts/dates.js.coffee</b>
<code>
$ ->
  $("input.datepicker").each (i) ->
    $(this).datepicker
      altFormat: "yy-mm-dd"
      dateFormat: "mm/dd/yy"
      altField: $(this).next()
</code>

Pretty straightforward. Loop through each input.datepicker, give it a user friendly format (mm/dd/yy), give it an alt format to send to rails (yy-mm-dd) and tell it to use the next sibling as the alt input. 

And that's all she wrote, you'll now have a nice datepicker field that is generic and can be utilized anywhere in your app. Enjoy!
