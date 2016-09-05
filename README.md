# registrationpages
This contains all pages registration related


HTML Form Guide 
Home
 Tutorials
 Downloads
 Making Awesome Forms
  





Creating a registration form using PHP


 Downloads, PHP Form 
 
 download, PHP, php registration form 
 

Creating a membership based site seems like a daunting task at first. If you ever wanted to do this by yourself, then just gave up when you started to think how you are going to put it together using your PHP skills, then this article is for you. We are going to walk you through every aspect of creating a membership based site, with a secure members area protected by password.

The whole process consists of two big parts: user registration and user authentication. In the first part, we are going to cover creation of the registration form and storing the data in a MySQL database. In the second part, we will create the login form and use it to allow users access in the secure area.


Download the code

You can download the whole source code for the registration/login system from the link below:
RegistrationForm.zip

Configuration & Upload

 The ReadMe file contains detailed instructions.

Open the source\include\membersite_config.php file in a text editor and update the configuration. (Database login, your website’s name, your email address etc). 

Upload the whole directory contents. Test the register.php by submitting the form.

The registration form


PHP registration form sample 

 In order to create a user account, we need to gather a minimal amount of information from the user. We need his name, his email address and his desired username and password. Of course, we can ask for more information at this point, but a long form is always a turn-off. So let’s limit ourselves to just those fields.

Here is the registration form:





<form id='register' action='register.php' method='post'
    accept-charset='UTF-8'>
<fieldset >
<legend>Register</legend>
<input type='hidden' name='submitted' id='submitted' value='1'/>
<label for='name' >Your Full Name*: </label>
<input type='text' name='name' id='name' maxlength="50" />
<label for='email' >Email Address*:</label>
<input type='text' name='email' id='email' maxlength="50" />
 
<label for='username' >UserName*:</label>
<input type='text' name='username' id='username' maxlength="50" />
 
<label for='password' >Password*:</label>
<input type='password' name='password' id='password' maxlength="50" />
<input type='submit' name='Submit' value='Submit' />
 
</fieldset>
</form>
 

So, we have text fields for name, email and the password. Note that we are using the password widget for better usability.

Form validation

At this point it is a good idea to put some form validation code in place, so we make sure that we have all the data required to create the user account. We need to check if name and email, and password are filled in and that the email is in the proper format.

We can use the free JavaScript form validation script to add form validations quickly and easily, with lesser code.

Here is a sample JavaScript validation code to be used for the sample form we created earlier:





var frmvalidator  = new Validator("register");
frmvalidator.EnableOnPageErrorDisplay();
frmvalidator.EnableMsgsTogether();
frmvalidator.addValidation("name","req","Please provide your name");
 
frmvalidator.addValidation("email","req","Please provide your email address");
 
frmvalidator.addValidation("email","email","Please provide a valid email address");
 
frmvalidator.addValidation("username","req","Please provide a username");
 
frmvalidator.addValidation("password","req","Please provide a password");
 

To be on the safe side, we will also have the same validations on the server side too. For server side validations, we will use the PHP form validation script

Handling the form submission

Now we have to handle the form data that is submitted.

Here is the sequence (see the file fg_membersite.php in the downloaded source):





function RegisterUser()
{
    if(!isset($_POST['submitted']))
    {
       return false;
    }
     
    $formvars = array();
     
    if(!$this->ValidateRegistrationSubmission())
    {
        return false;
    }
     
    $this->CollectRegistrationSubmission($formvars);
     
    if(!$this->SaveToDatabase($formvars))
    {
        return false;
    }
     
    if(!$this->SendUserConfirmationEmail($formvars))
    {
        return false;
    }
 
    $this->SendAdminIntimationEmail($formvars);
     
    return true;
}
 

First, we validate the form submission. Then we collect and ‘sanitize’ the form submission data (always do this before sending email, saving to database etc). The form submission is then saved to the database table. We send an email to the user requesting confirmation. Then we intimate the admin that a user has registered.

Saving the data in the database

Now that we gathered all the data, we need to store it into the database.
 Here is how we save the form submission to the database. 





function SaveToDatabase(&$formvars)
   {
       if(!$this->DBLogin())
       {
           $this->HandleError("Database login failed!");
           return false;
       }
       if(!$this->Ensuretable())
       {
           return false;
       }
       if(!$this->IsFieldUnique($formvars,'email'))
       {
           $this->HandleError("This email is already registered");
           return false;
       }
        
       if(!$this->IsFieldUnique($formvars,'username'))
       {
           $this->HandleError("This UserName is already used. Please try another username");
           return false;
       }        
       if(!$this->InsertIntoDB($formvars))
       {
           $this->HandleError("Inserting to Database failed!");
           return false;
       }
       return true;
   }
 

Note that you have configured the Database login details in the membersite_config.php file. Most of the cases, you can use “localhost” for database host.
 After logging in, we make sure that the table is existing.(If not, the script will create the required table).
 Then we make sure that the username and email are unique. If it is not unique, we return error back to the user.

The database table structure

This is the table structure. The CreateTable() function in the fg_membersite.php file creates the table. Here is the code:





function CreateTable()
{
    $qry = "Create Table $this->tablename (".
            "id_user INT NOT NULL AUTO_INCREMENT ,".
            "name VARCHAR( 128 ) NOT NULL ,".
            "email VARCHAR( 64 ) NOT NULL ,".
            "phone_number VARCHAR( 16 ) NOT NULL ,".
            "username VARCHAR( 16 ) NOT NULL ,".
            "password VARCHAR( 32 ) NOT NULL ,".
            "confirmcode VARCHAR(32) ,".
            "PRIMARY KEY ( id_user )".
            ")";
             
    if(!mysql_query($qry,$this->connection))
    {
        $this->HandleDBError("Error creating the table \nquery was\n $qry");
        return false;
    }
    return true;
}
 

The id_user field will contain the unique id of the user, and is also the primary key of the table. Notice that we allow 32 characters for the password field. We do this because, as an added security measure, we will store the password in the database encrypted using MD5. Please note that because MD5 is an one-way encryption method, we won’t be able to recover the password in case the user forgets it.

Inserting the registration to the table

Here is the code that we use to insert data into the database. We will have all our data available in the $formvars array.





function InsertIntoDB(&$formvars)
{
    $confirmcode = $this->MakeConfirmationMd5($formvars['email']);
 
    $insert_query = 'insert into '.$this->tablename.'(
            name,
            email,
            username,
            password,
            confirmcode
            )
            values
            (
            "' . $this->SanitizeForSQL($formvars['name']) . '",
            "' . $this->SanitizeForSQL($formvars['email']) . '",
            "' . $this->SanitizeForSQL($formvars['username']) . '",
            "' . md5($formvars['password']) . '",
            "' . $confirmcode . '"
            )';      
    if(!mysql_query( $insert_query ,$this->connection))
    {
        $this->HandleDBError("Error inserting data to the table\nquery:$insert_query");
        return false;
    }        
    return true;
}
 

Notice that we use PHP function md5() to encrypt the password before inserting it into the database.
 Also, we make the unique confirmation code from the user’s email address.

Sending emails

Now that we have the registration in our database, we will send a confirmation email to the user. The user has to click a link in the confirmation email to complete the registration process.





function SendUserConfirmationEmail(&$formvars)
{
    $mailer = new PHPMailer();
     
    $mailer->CharSet = 'utf-8';
     
    $mailer->AddAddress($formvars['email'],$formvars['name']);
     
    $mailer->Subject = "Your registration with ".$this->sitename;
 
    $mailer->From = $this->GetFromAddress();        
     
    $confirmcode = urlencode($this->MakeConfirmationMd5($formvars['email']));
     
    $confirm_url = $this->GetAbsoluteURLFolder().'/confirmreg.php?code='.$confirmcode;
     
    $mailer->Body ="Hello ".$formvars['name']."\r\n\r\n".
    "Thanks for your registration with ".$this->sitename."\r\n".
    "Please click the link below to confirm your registration.\r\n".
    "$confirm_url\r\n".
    "\r\n".
    "Regards,\r\n".
    "Webmaster\r\n".
    $this->sitename;
 
    if(!$mailer->Send())
    {
        $this->HandleError("Failed sending registration confirmation email.");
        return false;
    }
    return true;
}
 

We use the free PHPMailer script to send the email.
 Note that we make the confirmation URL point to confirmreg.php?code=XXXX (where XXXX is the confirmation code).
 In the confirmreg.php script, we search for this confirmation code and update the ‘confirmed’ field in the table.

After completing all these operations successfully, we send an email to the admin (configured in the membersite_config.php file)

See also:Making a login form using PHP

Updates

9th Jan 2012
 Reset Password/Change Password features are added
 The code is now shared at GitHub.

25th May 2011
 Now you can display the logged-in user’s name with this code:





Welcome back <?= $fgmembersite->UserFullName(); ?>!
 

License


 The code is shared under LGPL license. You can freely use it on commercial or non-commercial websites.


Be Sociable, Share!


   
   
   
