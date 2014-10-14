---
layout: post
title: MongoDB in the real world
created: 1300340266
---
With MongoDB 1.8 out, I'm seeing some renewed debate about it's place in modern web development. In general I've experienced 3 reactions when it comes to mongo...
<ol>
<li> Mongo is never an appropriate solution</li>
<li> Mongo is awesome for any project</li>
<li> Mongo is appropriate for some projects, but is probably a bit overhyped</li>
</ol>

For the record, I fall into the latter camp. The first group annoys me to no end, and in general I choose to ignore folks who will tell you that any specific tool is never appropriate (unless they are perhaps talking about ASP - joking!). The second group, well I don't agree, but they're Mostly Harmless. 

Mongo inspires this sort of debate because, unlike a lot of the other trendy NoSQL databases, it operates on a different principle. Most of your other solutions (Redis, Cassandra, CouchDB) are essentially key/value stores. They work great for things like caching, or storing data that's easily converted to a key/value format. Mongo, on the other hand, is a document store. Rather than being simply limited to key/value storage, it supports storing what amounts to a binary version of JSON. It also supports embedding documents, relations between documents and SQL like querying capability (including awesome map/reduce functionality). This all means that mongo can basically replace an SQL database, which upsets a lot of folks who feel that you shouldn't replace something proven and stable with some relative newcomer that could burn your app up in a blaze of glory.

The problem these fine people fail to consider is that not everything in the world fits well into a traditional SQL relational model, nor into a key/value model. Mongo allows you to store documents, which are collections of keys and values of various types (hash, array, integer, string, etc) as well as child documents. This makes it rather unique among the NoSQL offerings and allow it to solve a unique sort of problem. This post, hopefully, will explain one such problem and why I feel that Mongo is an awesome tool to have in your arsenal.

<h2>The Problem</h2>

Recently, I had an app whose core functionality was quite tricky to implement in a traditional manner. The app in question involved collecting data through a web interface, and then mapping that data to a significant number of PDF files. The app had a central form which data was entered into, as well as web interfaces for each individual PDF to edit data that was not on the master form, or change data on an individual form basis. 

When I first started, I worked as I usually do by trying to visualize the data model before working on any code. And instantly, I was finding it problematic. From a traditional relational sql perspective, each PDF form had a few attributes like a name. It also had many sections, which had many fields. This all seems very straight forward and could easily be implemented with a traditional SQL database. The problem comes from the fact that the sheer number of forms and fields made it impossible to use a traditional sql table structure (at the very least, it would have been messy to do so). You'd have to do something where each form had it's own table, which may have hundreds of columns. UGLY. On top of that, it also means you have to make schema changes to support changed forms. It was obvious the only practical way to do this was to use a key/value table to store individual field data. This is not a horrible solution, however in this case each field also had quite a bit of metadata in addition to the value. Each field had to have a page number and X/Y coordinates to know where to put it on the pdf, it needed a key to map data to it from the master form, it needed a type attribute to determine what sort of input to render for it in the application, it needed formatting data to determine how to draw it on individual forms (for example, a phone number may be 555-555-5555 on one form but (555) 555 5555 on another form), it needed to potentially have a validator which could be unique to each form, etc. We're now no longer using a simple key/value table.

To complicate this, to present these fields through a web interface, we also need a lot of information on how to render and group them. For example, fields for parts of a name (first, last, middle) should all be rendered on one line or somehow grouped together. Fields might also be dependent, so for example if you check YES for one field that may enable a series of other fields. Now our SQL structure is getting REALLY complicated. Not to mention, getting the data for all fields in a form in one fell swoop is now much more complicated, and searching through the fields is also a problem. Plus, if a form is saved the writes to the key/value table are going to be on the order of hundreds, which could be problematic once the data set grows large.

Now, this isn't our first time doing this, so I looked into the solutions we've used on similar apps. The solutions we've employed in the past have been based around storing data in a key/value table and then having XML files for each PDF. Essentially, the XML file is used to tell the system how to render each field to the web interface and to the PDF. This is a workable solution, however from past experience I know that it's rather miserable to maintain. XML is quite horrible to read and without automated tools it becomes very very tedious. Not to mention, trying to spec out a DSL for your XML files becomes difficult due to the sheer number of fields and various ways to format them.

At this point I began to investigate alternate ways of solving this problem and decided MongoDB looked like a good candidate. Unlike SQL or other NoSQL solutions, Mongo allowed us to store representations of our PDF forms in a natural way, as a document. 

<h2>The Mongo Solution</h2>

MongoDB allowed us to model this data, in a way that is schemaless, easily updated and FAST. I created three models for mongo, Form, Section and Field. The rest of the app used a traditional SQL database. As with a traditional database, Form had many sections and Sections had many Fields. Both Sections and Fields were embedded documents, which means they were essentially stored within the Form entry. We also enabled versioning on these, which with Mongoid saves versions within the same document. To give you an idea of what this data looks like in Mongo, here's an example in Json format:

<code>
{
   "_id": ObjectId("4d42f298c1d656078900000f"),
   "created_at": "Fri, 28 Jan 2011 11: 45: 12 -0500",
   "form_id": 7,
   "form_sections": {
     "0": {
       "_id": ObjectId("4d42f298c1d6560789000010"),
       "name": "Applicant Details",
       "project_form_fields": {
         "0": {
           "_id": ObjectId("4d42f298c1d6560789000011"),
           "datatype": "string",
           "edited": 0,
           "label": "Name",
           "mapping": "company_name",
           "page": 1,
           "value": "Supernus Pharmaceuticals, Inc.",
           "xpos": "7",
           "ypos": "240"
           }
        }
      }
   }
}
</code>

Mongo models with Mongoid suppor the normal things you would expect in rails, relationships, named scopes, etc. However, they also contain fields. As mongo is schemaless, each model could essentially save whatever attribute you throw at it, so the fields give mongoid a way to know what to update and what type of data it may contain (by default all fields are strings). Here's the simple definition for the Form model:

<code>
class Form
  include Mongoid::Document
  include Mongoid::Timestamps
  include Mongoid::Versioning
  include Mongoid::Paranoia # soft delete
  
  field :name
  field :project_id, :type => Integer
  field :form_id, :type => Integer
  
  accepts_nested_attributes_for :form_sections
  
  validates :project_id, :presence => true
  validates :form_id, :presence => true
  
  embeds_many :form_sections
  
  belongs_to_related :project
  belongs_to_related :form
    
  scope :deleted, :where => { :deleted_at.exists => true }
end
</code>

Compared to a 'normal' ActiveRecord model, you'll note that we mention the fields and also have an embeds_many line which indicates that sections are included within this document. A few things to note about this...

<ul>
<li>You can work with fields which are not defined, however your standard rails accessors won't work. In other words, I could save a field 'foo' with a form, but I couldn't say form.foo = 'asdf'. Instead I'd have to do form['foo'] = 'asdf'. </li>
<li>You can't really relate mongo models to sql models. At least in the latter versions of Mongoid this has been removed, and I only ever got it working one way. This means that since my mongo form belongs to a project, I can't easily do Project.Form or Form.Project. 
</li>
</ul>

<h2>Using the Models</h2>

From here on out, working with mongo model is *almost* identical to a traditional sql model. If I want to find the form for Project 1 and State 15, I just issue:

<code>
Form.find({:project_id => 1, :state_id => 14})
</code>

In my case this returns a nested hash for the object. If I wanted to print out all the fields I could do:

<code>
form = Form.find(...)
form.sections.each do |section|
  section.fields.each do |field|
    puts field.name
  end
end
</code>

Now this model contains a lot of data, hundreds of fields. However, loading it all up is extremely quick since you are essentially fetching one indexed document. Since all the other data is embedded, you get it all in one fell swoop. Likewise with saving the data, just use your standard rails nested form helpers and pass the data to save. 

<h2>Building our models<h2>

One other problem we ran into was building out these models. We now have a way for each Form to save it's field values and all the needed meta-data to generate a PDF or HTML version of it. However, as you can imagine, generating these is REALLY tedious. Consider you now have 200 of these forms each with 100 fields, and you've got a complicated nested structure to build for each one of them. We needed an easy way to maintain these form structures and populate them for new entries. You'll note in the design that each form contains the full data for each project, and this is completely intentional. Each project needs to be isolated, such that if we update a form for next year, the projects which are archived should still work and in no way reference the new structure. Considering this, what we needed was a template.

I decided to implement this using a YAML format to model the structure of each form. It's essentially just a YAML version of the JSON format you see above, and gives us an easy to read and maintain format for each form. We also built a tool to let us generate the yaml, by loading up a PDF and letting us draw form controls on top of it. 

<h2>The workflow</h2>

To put it all together, let me describe the workflow of the application.

<ol>
<li>A new project is created through the web interface.</li>
<li>The project details are filled out, including many reusable data points that will be mapped to individual forms</li>
<li>Individual forms are 'enabled' through the web interface, depending on the project needs</li>
<li>When a form is enabled, a new MongoDB document is created and it is populated from the appropriate YAML definition </li>
<li>After the form is populated from YAML, each field's key is examined, and data matching that key from the project details is imported to it's value field</li>
<li>Each time the form is saved, a historical version is saved in the document</li>
<li>At any time, the user can choose to re-import data from the project details. Any fields which have not been edited are re-imported.</li>
<li>Since each project's forms contain all the data needed to render them, changes in the YAML structure have no impact. However, if needed, the forms can be upgraded to a newer version</li>
<li>When it's time to generate the PDF's, the fields are iterated through and printed out at the x and y coordinates they specify. In addition, formatters stored in document may be used to reformat them to match the PDF (for example, name may specify using a box formatter with a width and height, which prints out one letter per rectangular box on the form)</li>
</0l>

And that's about it!

<h2>Conclusion</h2>

While it's not all roses, I do believe in this instance Mongo gave us a much better solution than a traditional SQL database. It helped achieve the goals of having a flexible method of storing dynamic data in a way that we could easily change (for example, if we realize that fields need a new attribute we can easily add it with no schema changes). It made it easy to keep each form isolated from changes in the master form definitions as well as keep a complete version history. It is also extremely fast, and the code to work with it is very simple (code to create each entry from YAML is less than 15 lines, and the entire PDF generator is about 60 lines).  While it's certainly a problem that could be solved with SQL, it would have been far messier and much more difficult to maintain.
