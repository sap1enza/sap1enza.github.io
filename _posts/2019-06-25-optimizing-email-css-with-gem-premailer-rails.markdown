---
layout: post
title: Optimizing email CSS on rails with gem premailer
tags: [ruby, rails, improvements]
image: '/images/posts/email_marketing.png'
---

There are a few concerns when you develop an email marketing: use tables, take
care with the size of the images, send forms, pay attention at the compatibility
with older browsers or devices and avoid some tags, for example.

Resuming, in email clients the HTML is changed by a pre-processor before it gets
rendered, making a few changes which it worth to be aware of

* Remove or change HTML tags
* Remove or change CSS
* Change the entire structure of the document
* Remove the whole javascript

Taking `gmail` as example, it removs any tag `<link>` or `<style>` and change
the entire structure of the document, this force us to write code like this

```html5
<table id="customers" style="font-family: 'Trebuchet MS', Arial; border-collapse: collapse; width: 100%;">
  <tbody>
    <tr style="background-color: #f2f2f2;">
      <th style="text-align: left; background-color: #4caf50; color: white; border: 1px solid #ddd; padding: 8px;">Company</th>
      <th style="text-align: left; background-color: #4caf50; color: white; border: 1px solid #ddd; padding: 8px;">Contact</th>
      <th style="text-align: left; background-color: #4caf50; color: white; border: 1px solid #ddd; padding: 8px;">Country</th>
    </tr>
    <tr style="background-color: #f2f2f2;">
      <td style="border: 1px solid #ddd; padding: 8px;">Alfreds Futterkiste</td>
      <td style="border: 1px solid #ddd; padding: 8px;">Maria Anders</td>
      <td style="border: 1px solid #ddd; padding: 8px;">Germany</td>
    </tr>
    <tr>
      <td style="border: 1px solid #ddd; padding: 8px;">Berglunds snabbköp</td>
      <td style="border: 1px solid #ddd; padding: 8px;">Christina Berglund</td>
      <td style="border: 1px solid #ddd; padding: 8px;">Sweden</td>
    </tr>
    <tr style="background-color: #f2f2f2;">
      <td style="border: 1px solid #ddd; padding: 8px;">Centro comercial Moctezuma</td>
      <td style="border: 1px solid #ddd; padding: 8px;">Francisco Chang</td>
      <td style="border: 1px solid #ddd; padding: 8px;">Mexico</td>
    </tr>
  </tbody>
</table>
```

The problem is that it ruins the legibility and maintenance of the code,
just imagine a scenario a little bit more complex and the need of change the style
of a class, the result: the pandora's box has opened

An alternative to circumventing this and obtaining a file location in a separate
file without compromising file rendering on any mail server is to use the gem `premailer`

* https://github.com/premailer/premailer

It check the tags `style` and `link[rel=stylesheet]` and convert them into inline attributes,
besides converting relative url into absolute ones. To add into your project it's just
put `gem 'premailer'` at your `Gemfile` and runs `bundle install`.

To use the premailer when sending emails, you can pass a block with the desired format

```ruby
mail(
  to: 'abc@dfg.com',
  subject: 'Utilizando premailer'
) do |format|
  format.html do
    Premailer.new(
      render(template: 'caminho_do_template'),
      with_html_string: true
    ).to_inline_css
  end
end
```

Here i instantiate the class `Premailer` passing the `render` as argument and setting
the attribute `with_html_string` as true, because the render will return an string with
the html content, by default the premailer receives the file path, i need to explicify
that i will send the string content as argument, and finaly use the method `to_inline_css` which
will do all the magic.

This makes possible us to do something like

```html5
#customers {
  font-family: "Trebuchet MS", Arial, Helvetica, sans-serif;
  border-collapse: collapse;
  width: 100%;
}

#customers td, #customers th {
  border: 1px solid #ddd;
  padding: 8px;
}

#customers tr:nth-child(even){background-color: #f2f2f2;}

#customers th {
  padding-top: 12px;
  padding-bottom: 12px;
  text-align: left;
  background-color: #4CAF50;
  color: white;
}

<table id="customers">
  <tbody>
    <tr>
      <th>Company</th>
      <th>Contact</th>
      <th>Country</th>
    </tr>
    <tr>
      <td>Alfreds Futterkiste</td>
      <td>Maria Anders</td>
      <td>Germany</td>
    </tr>
    <tr>
      <td>Berglunds snabbköp</td>
      <td>Christina Berglund</td>
      <td>Sweden</td>
    </tr>
    <tr>
      <td>Centro comercial Moctezuma</td>
      <td>Francisco Chang</td>
      <td>Mexico</td>
    </tr>
  </tbody>
</table>
```

When the mail is sent, the premailer is executed and do its magic, returning
an code like the first example, the advantage is that whe mantaing the code legibility
at the same time that we allow a mantaining less harder, besides we uses a good practice
that is separate the css of the html.
