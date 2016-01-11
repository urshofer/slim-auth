Slim Auth <https://travis-ci.org/jeremykendall/slim-auth> <https://coveralls.io/r/jeremykendall/slim-auth?branch=master> <http://depending.in/jeremykendall/slim-auth>
======================================================================================================================================================================

Slim Auth is an authorization and authentication library for the [Slim
Framework](<http://slimframework.com/>). Authentication is provided by the Zend
Framework
[Zend\\Authentication](<http://framework.zend.com/manual/current/en/modules/zend.authentication.intro.html>)
component, and authorization by the Zend Framework
[Zend\\Permissions\\Acl](<http://framework.zend.com/manual/current/en/modules/zend.permissions.acl.intro.html>)
component.

Fair Warning: Documentation Mostly Complete
-------------------------------------------

Slim Auth is fully functional and production ready (I've used it in production
in multiple projects), but this documentation is incomplete. (Current status of
the documentation is \~90% complete.)

If you're familiar with
[Zend\\Authentication](<http://framework.zend.com/manual/current/en/modules/zend.authentication.intro.html>)
and
[Zend\\Permissions\\Acl](<http://framework.zend.com/manual/current/en/modules/zend.permissions.acl.intro.html>),
you'll be able to implement the library without any trouble. Otherwise, you
might want to wait for the docs to be completed (no ETA) or open a GitHub issue
with any questions or problems you encounter.

Caveat emptor and all that.

Slim SessionCookie No Longer Recomended
---------------------------------------

**TL;DR**: You *will* experience unexpected behavior if you use
`Zend\Authentication\Storage\Session` as your auth storage and
`Slim\Middleware\SessionCookie` to provide encrypted cookies when your Slim
version is \>= 2.6.

Earlier versions of this documentation (and the [sample
implementation](<https://github.com/jeremykendall/slim-auth-impl>)) demonstrated
the use of Slim's [SessionCookie
Middleware](<http://docs.slimframework.com/#Cookie-Session-Store>) as a way to
handle session storage in concert with Zend Session. As of [Slim
2.6.0](<https://github.com/slimphp/Slim/releases/tag/2.6.0>), Zend Session and
Slim's SessionCookie middleware no longer play well together, and I've opted for
a Zend Session only approach.

Requirements
------------

Slim Auth works with all versions of Slim 2 \>= 2.4.2. Slim Auth has not been
tested against the upcoming Slim 3 release.

Example Implementation
----------------------

I've put together an example implementation to demonstrate the library in
action. The example implementation can be found
[here](<https://github.com/jeremykendall/slim-auth-impl>).

Installation
------------

Installation is provided via [Composer](<http://getcomposer.org>).

First, install Composer.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
curl -s https://getcomposer.org/installer | php
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Then install Slim Auth with the following Composer command.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
composer require jeremykendall/slim-auth
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Finally, add this line at the top of your application’s index.php file:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
require 'vendor/autoload.php';
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Preparing Your App For Slim Auth
--------------------------------

### Database

Your database should have a user table, and that table must have a `role`
column. The contents of the `role` column should be a string and correspond to
the roles in your ACL. The table name and all other column names are up to you.

Here's an example schema for a user table. If you don't already have a user
table, feel free to use this one:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
CREATE TABLE IF NOT EXISTS [users] (
    [id] INTEGER NOT NULL PRIMARY KEY,
    [username] VARCHAR(50) NOT NULL,
    [role] VARCHAR(50) NOT NULL,
    [password] VARCHAR(255) NULL
);
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

### ACL

An Access Control List, or ACL, defines the set of rules that determines which
group of users have access to which routes within your Slim application. Below
is a very simple example ACL. Please pay special attention to the comments.

*Please refer to the* [Zend\\Permissions\\Acl
documentation](<http://framework.zend.com/manual/current/en/modules/zend.permissions.acl.intro.html>)
*for complete details on using the Zend Framework ACL component.*

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
namespace Example;

use Zend\Permissions\Acl\Acl as ZendAcl;

class Acl extends ZendAcl
{
    public function __construct()
    {
        // APPLICATION ROLES
        $this->addRole('guest');
        // member role "extends" guest, meaning the member role will get all of 
        // the guest role permissions by default
        $this->addRole('member', 'guest');
        $this->addRole('admin');

        // APPLICATION RESOURCES
        // Application resources == Slim route patterns
        $this->addResource('/');
        $this->addResource('/login');
        $this->addResource('/logout');
        $this->addResource('/member');
        $this->addResource('/admin');

        // APPLICATION PERMISSIONS
        // Now we allow or deny a role's access to resources. The third argument
        // is 'privilege'. We're using HTTP method as 'privilege'.
        $this->allow('guest', '/', 'GET');
        $this->allow('guest', '/login', array('GET', 'POST'));
        $this->allow('guest', '/logout', 'GET');

        $this->allow('member', '/member', 'GET');

        // This allows admin access to everything
        $this->allow('admin');
    }
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

#### The Guest Role

Please note the `guest` role. **You must use the name** `guest` **as the role
assigned to unauthenticated users**. The other role names are yours to choose.

#### Acl "Privileges"

**IMPORTANT**: The third argument to `Acl::allow()`, 'privileges', is either a
string or an array, and should be an HTTP verb or HTTP verbs respectively. By
adding the third argument, you are restricting route access by HTTP method. If
you do not provide an HTTP verb or verbs, you are allowing access to the
specified route via *all* HTTP methods. **Be extremely vigilant here.** You
wouldn't want to accidentally allow a 'guest' role access to an admin `DELETE`
route simply because you forgot to explicitly deny the `DELETE` route.

Configuring Slim Auth: Defaults
-------------------------------

Now that you have a user database table with a `role` column and an ACL, you're
ready to configure Slim Auth and add it to your application.

First, add `use` statements for the PasswordValidator (from the [Password
Validator](<https://github.com/jeremykendall/password-validator>) library), the
PDO adapter, and the Slim Auth Bootstrap.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
use JeremyKendall\Password\PasswordValidator;
use JeremyKendall\Slim\Auth\Adapter\Db\PdoAdapter;
use JeremyKendall\Slim\Auth\Bootstrap;
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Next, create your Slim application.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$app = new \Slim\Slim();
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

### Authentication Adapter

From the Zend Authentication documentation:

>   `Zend\Authentication` adapters are used to authenticate against a particular
>   type of authentication service, such as LDAP, RDBMS, or file-based storage.

Slim Auth provides an RDBMS authentication adapter for
[PDO](<http://php.net/manual/en/book.pdo.php>). The constructor accepts five
required arguments:

-   A `\PDO` instance

-   The name of the user table

-   The name of the identity, or username, column

-   The name of the credential, or password, column

-   An instance of `JeremyKendall\Password\PasswordValidator`

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$db = new \PDO(<database connection info>);
$adapter = new PdoAdapter(
    $db, 
    <user table name>, 
    <identity column name>, 
    <credential column name>,
    <role column name>, 
    new PasswordValidator()
);
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

>   **NOTE**: Please refer to the [Password Validator
>   documentation](<https://github.com/jeremykendall/password-validator>) for
>   more information on the proper use of the library. If you choose not to use
>   the Password Validator library, you will need to create your own
>   authentication adapter.

### Putting it all Together

Now it's time to instantiate your ACL and bootstrap Slim Auth.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$acl = new \Namespace\For\Your\Acl();
$authBootstrap = new Bootstrap($app, $adapter, $acl);
$authBootstrap->bootstrap();
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

### Login Route

You'll need a login route, of course, and it's important that you name your
route `login` using Slim's [Route
Names](<http://docs.slimframework.com/#Route-Names>) feature.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$app->map('/login', function() {})->via('GET', 'POST')->name('login');
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This allows you to use whatever route pattern you like for your login route.
Slim Auth will redirect users to the correct route using Slim's `urlFor()`
[Route Helper](<http://docs.slimframework.com/#Route-Helpers>).

Here's a sample login route:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
// Login route MUST be named 'login'
$app->map('/login', function () use ($app) {
    $username = null;

    if ($app->request()->isPost()) {
        $username = $app->request->post('username');
        $password = $app->request->post('password');

        $result = $app->authenticator->authenticate($username, $password);

        if ($result->isValid()) {
            $app->redirect('/');
        } else {
            $messages = $result->getMessages();
            $app->flashNow('error', $messages[0]);
        }
    }

    $app->render('login.twig', array('username' => $username));
})->via('GET', 'POST')->name('login');
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

### Logout Route

As authentication stores the authenticated user's identity, logging out consists
of nothing more than clearing that identity. Clearing the identity is handled by
`Authenticator::logout`.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$app->get('/logout', function () use ($app) {
    $app->authenticator->logout();
    $app->redirect('/');
});
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

And Done
--------

That should get you most of the way. I'll complete documentation as soon as I'm
able, but can't currently commit to an ETA. Again, please feel free to open and
issue with any questions you might have regarding implementation.

Thanks for considering Slim Auth for your project.
