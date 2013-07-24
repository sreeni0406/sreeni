# Apache Shiro plugin for Stormpath #

The `stormpath-shiro` plugin allows an [Apache Shiro](http://shiro.apache.org)-enabled application
use the [Stormpath](http://www.stormpath.com) User Management & Authentication service for all authentication and access control needs.

Pairing Shiro with Stormpath gives you a full application security system complete with immediate user account
support, authentication, account registration and password reset workflows, password security and more -
with little to no coding on your part.

## Configuration ##

1. Add the stormpath-shiro .jars to your application using Maven, Ant+Ivy, Grails, SBT or whatever
   maven-compatible tool you prefer:

        <dependency>
            <groupId>com.stormpath.shiro</groupId>
            <artifactId>stormpath-shiro-core</artifactId>
            <version>0.3.1</version>
        </dependency>
        <dependency>
            <groupId>com.stormpath.sdk</groupId>
            <artifactId>stormpath-sdk-httpclient</artifactId>
            <version>0.8.0</version>
            <scope>runtime</scope>
        </dependency>

2. Ensure you [have an API Key](http://www.stormpath.com/docs/quickstart/connect) so your application can communicate
   with Stormpath.  Store your API Key file somewhere secure (readable only by you), for example:

        /home/myhomedir/.stormpath/apiKey.properties

3. Configure `shiro.ini` with the Stormpath `ApplicationRealm`:

        [main]
        ...
        stormpathClient = com.stormpath.shiro.client.ClientFactory
        # Replace this value with the file location from #2 above:
        stormpathClient.clientBuilder.apiKeyFileLocation = /home/myhomedir/.stormpath/apiKey.properties

        stormpathRealm = com.stormpath.shiro.realm.ApplicationRealm
        stormpathRealm.client = $stormpathClient
        stormpathRealm.applicationRestUrl = REPLACE_ME_WITH_YOUR_STORMPATH_APP_REST_URL

        securityManager.realm = $stormpathRealm

4. Replace the `stormpathRealm.applicationRestUrl` value above with your
   [Application's Stormpath-specific REST URL](http://www.stormpath.com/docs/libraries/application-rest-url), for
   example:

        stormpathRealm.applicationRestUrl = https://api.stormpath.com/v1/applications/someRandomIdHereReplaceMe

## Authentication ##

In a web application, you can use one of Shiro's existing authentication filters to automatically handle authentication requests (e.g. [BasicHttpAuthenticationFilter](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authc/BasicHttpAuthenticationFilter.html), [FormAuthenticationFilter](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authc/FormAuthenticationFilter.html)) and you won't have to code anything - authentication attempts will be processed as expected by the `StormpathRealm` automatically.

However, if you want to execute the authentication attempt yourself (e.g. you have a more complex login form or UI technology) or your application is not a web application, this is easy as well:

Create a Shiro `UsernamePasswordToken` to wrap your user's submitted username and password and then call Shiro's `Subject.login` method:

    String username = //get from a form or request parameter (OVER SSL!)
    String password = //get from a form or request parameter (OVER SSL!)
    String host = //end-user's host IP for auditing, e.g. servletRequest.getRemoteHost();

    UsernamePasswordToken token = new UsernamePasswordToken(username, password, host);

    Subject currentUser = SecurityUtils.getSubject();
    currentUser.login(token);

that's it - a standard Shiro authentication attempt.  If the authentication attempt fails, an `AuthenticationException` will be thrown as expected.

In Stormpath, you can add, remove and enable accounts for your application and Shiro reflects these changes instantly!

## Authorization ##

After an account has authenticated, you can perform standard Shiro role and permission checks, e.g. `subject.hasRole(roleName)` and `subject.isPermitted(permission)`.

### Role checks ###

Shiro's role concept in Stormpath is represented as a Stormpath [Group](http://www.stormpath.com/docs/java/product-guide#Groups).

#### Role checks with the Group `href` ####

The recommended way to perform a Shiro role check is to use the Stormpath group's `href` property as the Shiro role 'name'.  

While it is possible (and maybe more intuitive) to use the Group name for the role check, this secondary approach is not enabled by default and not recommended for most usages:  role names can potentially change over time (for example, someone changes the Group name in the Stormpath administration console without telling you).  If you code a role check in your source code, and that role name changes in the future, your role checks will likely fail!

Instead, it is recommended to perform role checks with a stable identifier.

You can use a Stormpath Group's `href` property as the role 'name' and check that:

    String groupHref = stormpathGroup.getHref();
    if (subject.hasRole(groupHref)) { 
        //do something 
    }

#### Role checks with the Group `name` ####

If you still want to use a Stormpath Group's name as the Shiro role name for role checks - perhaps because you have a high level of confidence that no one will change group names once your software is written - you can still use the Group name if you wish by adding a little configuration.

In your `shiro.ini` (or compatible configuration mechanism), you can set the supported naming modes of what will be represented as a Shiro role:

    [main]
    ...
    groupRoleResolver = com.stormpath.shiro.realm.DefaultGroupRoleResolver
    groupRoleResolver.setModeNames = NAME
    stormpathRealm.groupRoleResolver = $groupRoleResolver

The modes (or mode names) allow you to specify which Group properties Shiro will consider as role 'names'.  The default is `href`, but you can specify more than one if desired.  The supported modes are the following:

* *HREF*: the Group's `href` property will be considered a Shiro role name.  This is the default mode if not configured otherwise.  Allows a Shiro role check to look like the following: `subject.hasRole(group.getHref())`.
* *NAME*: the Group's `name` property will be considered a Shiro role name.  This allows a Shiro role check to look like the following: `subject.hasRole(group.getName())`.  This however has the downside that if you (or someone else on your team or in your company) changes the Group's name, you will have to update your role check code to reflect the new names (otherwise the existing checks are very likely to fail).
* *ID*: the Group's unique id will be considered a Shiro role name.  The unique id is the id at the end of the Group's HREF url.  This is a deprecated mode and should ideally not be used in new applications.

#### The GroupRoleResolver Interface ####

If the above default role name resolution logic does not meet your needs or if you want full customization of how a Stormpath Group resolves to one or more Shiro role names, you can implement the `GroupRoleResolver` interface and configure the implementation on the StormpathRealm:

    [main]
    ...
    groupRoleResolver = com.mycompany.my.impl.MyGroupRoleResolver
    stormpathRealm.groupRoleResolver = $groupRoleResolver

### Permission checks ###

Stormpath does not yet have a permission concept that maps to Shiro's, but you can still resolve a set of Permissions that you want attributed with a Stormpath Group or Stormpath Account by implementing the  `*PermissionResolver` interfaces.  This allows you to call Shiro `subject.isPermitted` methods and have them function correctly based on Stormpath Group or Accounts.

The `*PermissionResolver` interfaces allow you to simulate full permission associations to Stormpath `Group`s or `Account`s even though Stormpath does not store these permissions directly.  Your `*PermissionResolver` implementation(s) would likely query a data store or file or other mechanism to get assigned permissions for a given Stormpath `Group` or `Account`.

##### How `StormpathRealm` Permission Checks Work #####

The `StormpathRealm` will use any configured `AccountPermissionResolver` and `GroupPermissionResolver` instances to create the aggregate of all permissions attributed to a `Subject` during a permission check.  In other words, the following call:

    subject.isPermitted(aPermission)

will return `true` if the following is true:

* any of the permissions returned by the `AccountPermissionResolver` for the Subject's backing Account implies `aPermission`
* any of the permissions returned by the `GroupPermissionResolver` for any of the backing Account's Groups implies `aPermission`

`false` will be returned if `aPermission` is not implied by any of these permissions.

For further clarity, the `isPermitted` check works something like this (simplified for brevity):

    Set<Permission> accountPermissions = accountPermissionResolver.resolvePermissions(account);
    for (Permission accountPermission : accountPermissions) {
        if (accountPermission.implies(permissionToCheck)) {
            return true;
        }
    }
 
    for (Group group : account.getGroups()) {
        Set<Permission> groupPermissions = resolvePermissions(group);
        for (Permission groupPermission : groupPermissions) {
            if (groupPermission.implies(permissionToCheck)) {
                return true;
            }
        }
    }

    //otherwise not permitted:
    return false;

#### AccountPermissionResolver ####

The StormpathRealm's `AccountPermissionResolver` inspects a Stormpath `Account` and returns a set of Shiro `Permission`s that are considered directly assigned to that `Account`.

This interface is provided to resolve permissions that are _directly_ assigned to a Stormpath `Account`.  Permissions that are assigned to an account's groups (and therefore implicitly or indirectly associated with an `Account`) are best provided by a `GroupPermissionResolver` instance instead.

Your `AccountPermissionResolver` implementation could then be configured on the StormpathRealm instance.  For example, in `shiro.ini`:

    [main]
    ...
    accountPermissionResolver = com.mycompany.stormpath.shiro.MyAccountPermissionResolver
    stormpathRealm.accountPermissionResolver = $accountPermissionResolver

After you've configured this you can perform permission checks.  For example, perhaps you want to check if the current account is allowed to update their own information:

    String updateSelf = "account:" + subject.getPrincipal() + ":update";
    if (subject.isPermitted(updateSelf)) {
        //do something
    }

This check would succeed if the `MyAccountPermissionResolver` implementation returned that permission for the Subject's backing `Account`.

#### GroupPermissionResolver ####

The StormpathRealm's `GroupPermissionResolver` inspects a Stormpath `Group` and returns a set of Shiro `Permission`s that are considered assigned to that `Group`.

You can configure a custom `GroupPermissionResolver` implementation on the StormpathRealm instance.  For example, in `shiro.ini`:

    [main]
    ...
    groupPermissionResolver = com.mycompany.stormpath.shiro.MyGroupPermissionResolver
    stormpathRealm.groupPermissionResolver = $groupPermissionResolver

After you've configured this you can perform group permission checks.  For example, perhaps you want to check if the current Subject is allowed to edit a specific blog article:

    String editArticle = "blogArticle:" + article.getId() + ":edit";
    if (subject.isPermitted(editArticle)) {
        //do something
    }

This check would succeed if the Subject's direct permissions _or any of its Groups' permissions_ (as returned by the `MyGroupPermissionResolver` implementation) implied the 'editArticle` permission.