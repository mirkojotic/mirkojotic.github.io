---
title: 'Laravel: Conditional Validation of Arrays and Nested Items'
date: '2018-05-08 16:13:11'
---

One of the features Laravel comes bundled with is validation. [`Validator`](https://laravel.com/api/4.2/Illuminate/Validation/Validator.html) comes with Laravel and it is amazing (IMHO)! Take it from someone who has had to validate input in a lot of frameworks and languages.

A lot of what Laravel `Validator` can do is already in docs, I will just provide a semi-real-world example.

I will start with a fresh Laravel install (5.6.\*).

First let's set up this semi-real-world scenario. We have a page where user can select how he wants to be notified of something. User can pick between three options - email, phone or mail ( you know letters in envelopes... nobody said that the example has to make sense :D ). User can select either, but depending on what they select we want to make sure that they actually provide proper data. So if a user selects email as his preference we want them to input that email. In addition to that user can add more than one prefered contact.

`routes/web.php`
```php

Route::post('/notifyme', 'NotificationController@notifyme');

```

First we add a route that will handle users submission. After that we create `NotificationController` and add a `notifyme` method.

```php
<?php
// app/Http/Controllers/NotificationController.php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class NotificationController extends Controller
{
    public function notifymebulk(Request $request) {

      $request->validate([
        'contact' => 'required|in:email,phone,mail',
        'email.*'   => 'required_if:contact,email',
        'phone.*'   => 'required_if:contact,phone',
        'mail.*'    => 'required_if:contact,mail',
      ]);

      return view('notifyme.success'); // non existant view file
    }
}     

```

Just for reference this is what our form would look like:
```html
<form action="/notifyme" method="POST">
	<select name="contact" value="">
		<option value="">Please select your notification preference.</option>
		<option value="email">E-mail</option>
		<!-- other options -->
	</select>
	<!-- other input -->
	<button type="submit" value="Submit" />
</form>
```

The way this validation works is that array keys designate input values that are being validated.

In our case we want to make sure that value of select is either email, phone or mail.

We do that with `'contact' => 'required|in:email,phone,mail',`. Laravel will check if value of contact select element is one the comma separated strings we provided.

Where it gets fun is with conditional validation. We want to make sure emails are present if value of contact is 'email'. Since user can input more than one email we will use some more laravel magic.

If a key of array passed to validator is `email.*`, validator will treat `email` entry in POST as if it is an array and apply validation rules to **each** element in that `email` array.

Since we want to make it dependent on whatever user selects we use `required_if`. I think of `required_if` as a function that accepts two arguments (`anotherfield,value` - borrowed from Laravel docs). It roughtly translates to code like this:

```php
if($request->input('anotherfield', null) == value) {
	// apply validation
}
```

This POST payload will fail validation:
```php
[
	'contact' => 'email',
	'email' => ['']
]
```

Now this will work just fine but our error messages will not be as friendly as if we only had a single input field.

`The email.0 field is required when contact is email.`

We can't blame Laravel for this because it has no other way to phrase this. This is actually helpfull if you want to determine which array item is invalid but if you want to just simply display errors to user there is a quick way to format this error message.

Second parameter that validator accepts is an array to customize these messages.

```php
$request->validate(
	[
		'contact' => 'required|in:email,phone,mail',
		'email.*'   => 'required_if:contact,email',
		'phone.*'   => 'required_if:contact,phone',
		'mail.*'    => 'required_if:contact,mail',
	], 
	[
		'email.*' => 'If your prefered contact is email then you must input at least one email address.',
		'phone.*' => 'If your prefered contact is phone then you must input at least one phone number.',
		'mail.*' => 'If your prefered contact is mail then you must input at least one mailing address.',
	]
);
```

If you omit any of the input fields you are validating and it fails, the validation message will be the default one.


A quick side note:
If you have multiple rules for one field, you can customize validation error message for each rule:

```php
$request->validate(
	[
		'name' => 'required|min:2|max:30',
	], 
	[
		'name.required' => "Hey, don't you want to tell us your name?",
		'name.min:2' => "What kind of a name has less than 2 characters? C'mon!.",
		'name.max:30' => "OK. Let's not get carried away, just give us a TL;DR.",
	]
);
```
