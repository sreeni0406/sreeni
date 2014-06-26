# Apache Shiro plugin for Stormpath #

The Apache Shiro plugin for Stormpath allows an [Apache Shiro](http://shiro.apache.org)-enabled application to easily use the [Stormpath](http://www.stormpath.com) User Management & Authentication service for all authentication and access control needs.

Pairing Shiro with Stormpath gives you a full application security system complete with immediate user account
support, authentication, account registration and password reset workflows, password security and more -
with little to no coding on your part.

*Table of Contents*

- [Configuration](#configuration)
- [Authentication](#authentication)
- [Authorization](#authorization)
    - [Roles](#roles)
        - [Assigning Roles](#assigning-roles)
        - [Checking Roles](#checking-roles)
    - [Permissions](#permissions)
        - [Assigning Permissions](#assigning-permissions)
        - [Checking Permissions](#checking-permissions)
        - [Permission Storage](#permission-storage)
        - [How Permission Checks Work](#how-permission-checks-work)
- [Caching](#caching)

## Configuration ##

1. Add the stormpath-shiro .jars to your application using Maven, Ant+Ivy, Grails, SBT or whatever maven-compatible tool you prefer:
    
    ```xml
    <dependency>
      <groupId>com.stormpath.shiro</groupId>
      <artifactId>stormpath-shiro-core</artifactId>
      <version>0.6.0</version>
    </dependency>
    <dependency>
      <groupId>com.stormpath.sdk</groupId>
      <artifactId>stormpath-sdk-httpclient</artifactId>
      <version>1.0.RC2</version>
      <scope>runtime</scope>
    </dependency>
    ```
2. Ensure you [have an API Key](http://docs.stormpath.com/rest/quickstart) so your application can communicate
   with Stormpath.  Store your API Key file somewhere secure (readable only by you), for example:

    ```text
    /home/myhomedir/.stormpath/apiKey.properties
    ```

3. Configure `shiro.ini` with the Stormpath `ApplicationRealm`:
    ```ini
    [main]
    
    stormpathClient = com.stormpath.shiro.client.ClientFactory
    ; Replace this value with the file location from #2 above:
    stormpathClient.apiKeyFileLocation = /home/myhomedir/.stormpath/apiKey.properties
    ; If you've configured a Shiro CacheManager (recommended to reduce network calls):
    stormpathClient.cacheManager = $cacheManager
    
    stormpathRealm = com.stormpath.shiro.realm.ApplicationRealm
    stormpathRealm.client = $stormpathClient
    stormpathRealm.applicationRestUrl = REPLACE_ME_WITH_YOUR_STORMPATH_APP_REST_URL
    
    securityManager.realm = $stormpathRealm
    ```
4. Replace the `stormpathRealm.applicationRestUrl` value above with your [Application's Stormpath-specific REST URL](http://docs.stormpath.com/rest/product-guide/#locate-an-applications-rest-url), for example:
    
    ```ini
    stormpathRealm.applicationRestUrl = https://api.stormpath.com/v1/applications/someRandomIdHereReplaceMe
    ```

## Authentication ##

In a web application, you can use one of Shiro's existing authentication filters to automatically handle authentication requests (e.g. [BasicHttpAuthenticationFilter](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authc/BasicHttpAuthenticationFilter.html), [FormAuthenticationFilter](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authc/FormAuthenticationFilter.html)) and you won't have to code anything - authentication attempts will be processed as expected by the `StormpathRealm` automatically.

However, if you want to execute the authentication attempt yourself (e.g. you have a more complex login form or UI technology) or your application is not a web application, this is easy as well:

Create a Shiro `UsernamePasswordToken` to wrap your user's submitted username and password and then call Shiro's `Subject.login` method:

```java
String username = //get from a form or request parameter (OVER SSL!)
String password = //get from a form or request parameter (OVER SSL!)
String host = //end-user's host IP for auditing, e.g. servletRequest.getRemoteHost();

UsernamePasswordToken token = new UsernamePasswordToken(username, password, host);

Subject currentUser = SecurityUtils.getSubject();
currentUser.login(token);
```
that's it - a standard Shiro authentication attempt.  If the authentication attempt fails, an `AuthenticationException` will be thrown as expected.

In Stormpath, you can add, remove and enable accounts for your application and Shiro reflects these changes instantly!

## Authorization ##

After an account has authenticated, you can perform standard Shiro role and permission checks, e.g. `subject.hasRole(roleName)` and `subject.isPermitted(permission)`.

### Roles ###

Shiro's role concept in Stormpath is represented as a Stormpath [Group](http://www.stormpath.com/docs/java/product-guide#Groups).

#### Assigning Roles

Because Shiro roles are represented as Groups in Stormpath, you assign a role to an account simply by adding an account to a group (or by adding a group to an account, depending on how you look at it).  For example:

```java
Account account = client.getResource(accountHref, Account.class);
Group group = client.getResource(groupHref, Group.class);

//assign the account to the group:
group.addAccount(account);

//this would have achieved the same thing:
//    account.addGroup(group);
```

#### Checking Roles

##### Role checks with the Group `href`

The recommended way to perform a Shiro role check is to use the Stormpath group's `href` property as the Shiro role 'name'.  

While it is possible (and maybe more intuitive) to use the Group name for the role check, this secondary approach is not enabled by default and not recommended for most usages:  role names can potentially change over time (for example, someone changes the Group name in the Stormpath administration console without telling you).  If you code a role check in your source code, and that role name changes in the future, your role checks will likely fail!

Instead, it is recommended to perform role checks with a stable identifier.

You can use a Stormpath Group's `href` property as the role 'name' and check that:
```java
String groupHref = stormpathGroup.getHref();
if (subject.hasRole(groupHref)) { 
    //do something 
}
```

##### Role checks with the Group `name`

If you still want to use a Stormpath Group's name as the Shiro role name for role checks - perhaps because you have a high level of confidence that no one will change group names once your software is written - you can still use the Group name if you wish by adding a little configuration.

In your `shiro.ini` (or compatible configuration mechanism), you can set the supported naming modes of what will be represented as a Shiro role:

```ini
[main]
; ... continued ...
groupRoleResolver = com.stormpath.shiro.realm.DefaultGroupRoleResolver
groupRoleResolver.setModeNames = NAME
stormpathRealm.groupRoleResolver = $groupRoleResolver
```

The modes (or mode names) allow you to specify which Group properties Shiro will consider as role 'names'.  The default is `href`, but you can specify more than one if desired.  The supported modes are the following:

* *HREF*: the Group's `href` property will be considered a Shiro role name.  This is the default mode if not configured otherwise.  Allows a Shiro role check to look like the following: `subject.hasRole(group.getHref())`.
* *NAME*: the Group's `name` property will be considered a Shiro role name.  This allows a Shiro role check to look like the following: `subject.hasRole(group.getName())`.  This however has the downside that if you (or someone else on your team or in your company) changes the Group's name, you will have to update your role check code to reflect the new names (otherwise the existing checks are very likely to fail).
* *ID*: the Group's unique id will be considered a Shiro role name.  The unique id is the id at the end of the Group's HREF url.  This is a deprecated mode and should ideally not be used in new applications.

#### The GroupRoleResolver Interface ####

If the above default role name resolution logic does not meet your needs or if you want full customization of how a Stormpath Group resolves to one or more Shiro role names, you can implement the `GroupRoleResolver` interface and configure the implementation on the StormpathRealm:

```ini
[main]
; ... continued ...
groupRoleResolver = com.mycompany.my.impl.MyGroupRoleResolver
stormpathRealm.groupRoleResolver = $groupRoleResolver
```

### Permissions ###

The 0.5.0 release of the Apache Shiro plugin for Stormpath enabled the ability to assign ad-hoc sets of permissions directly to Stormpath Accounts or Groups using the accounts' or groups' [Custom Data](http://docs.stormpath.com/rest/product-guide/#custom-data) resource.

Once assigned, the Stormpath `ApplicationRealm` will automatically check account and group `CustomData` for permissions to perform Shiro permission checks.

#### Assigning Permissions

The easiest way to assign permissions to an account or group is to get the account or group's `CustomData` resource and use the Shiro Stormpath plugin's `CustomDataPermissionsEditor` to assign or remove permissions.  The following example uses both the Stormpath Java SDK API and the Shiro Stormpath plugin API:

```java
//Instantiate an account (this is the normal Stormpath Java SDK API):
Account acct = client.instantiate(Account.class);
String password = "Changeme1!";
acct.setUsername("jsmith");
acct.setPassword(password);
acct.setEmail("jsmith@nowhere.com");
acct.setGivenName("Joe");
acct.setSurname("Smith");
    
//Now let's add some Shiro permissions to the account's customData:
//(this class is in the Shiro Stormpath Plugin API):
new CustomDataPermissionsEditor(acct.getCustomData())
    .append("user:1234:edit")
    .append("report:create")
    
//Add the new account with its custom data to an application (normal Stormpath Java SDK API):
acct = anApplication.createAccount(Accounts.newCreateRequestFor(acct).build());
```
You can assign permissions to a Group too:

```java
Group group = client.instantiate(Group.class);
group.setName("Users");
new CustomDataPermissionsEditor(group.getCustomData()).append("user:login");
group = anApplication.createGroup(group)
```

You might want to assign that account to the group.  *Any permissions assigned to a group are automatically inherited by accounts in the group*:

```java
group.addAccount(acct);
```

This is very convenient: You can assign permissions to many accounts simultaneously by simply adding them once to a group that the accounts share.  In doing this, the Stormpath `Group` is acting much more like a Shiro role.

That means, that if the `jsmith` account logs in, you can perform the following Shiro permission check:

```java
subject.isPermitted("user:login");
```

And this would return `true`, because, while `user:login` isn't directly assigned to the account, it *is* assigned to one of the account's groups.  Very nice.

#### Checking Permissions

There is nothing special here - you check permissions as you would normally using Shiro:

```java
subject.isPermitted("whatever:here");
```

The Stormpath `ApplicationRealm` will automatically know how to determine the permissions assigned to the account to help Shiro give a `true` or `false` answer.  The next sections cover the storage and retrieval details in case you're curious how it works, or if you'd like to customize the behavior or `CustomData` field name.

#### Permission Storage

The `CustomDataPermissionsEditor` shown above, and the Shiro Stormpath `ApplicationRealm` default implementation assumes that a default field named `apacheShiroPermissions` in an account's or group's `CustomData` resource can be used to store permissions assigned directly to the account or group.  This implies the `CustomData` resource's JSON would look something like this:

```json
{
    "apacheShiroPermissions": [
        "perm1",
        "perm2",
        "permN"
    ]
}
```

If you wanted to change the name to something else, you could specify the `setFieldName` property on the `CustomDataPermissionsEditor` instance:

```java
new CustomDataPermissionsEditor(group.getCustomData())
    .setFieldName("whateverYouWantHere")
    .append("user:login");
```
and this would result in the following JSON structure instead:

```json
{
    "whateverYouWantHere": [
        "user:login",
    ]
}
```

But *NOTE*: While the `CustomDataPermissionsEditor` implementation will modify the field name you specify, the, `ApplicationRealm` needs to read that same field during permission checks.  So if you change it as shown above, you must also change the realm's configuration to reference the new name as well:

```ini
[main]
; ... continued ...
stormpathRealm.groupPermissionResolver.customDataFieldName = whateverYouWantHere
stormpathRealm.accountPermissionResolver.customDataFieldName = whateverYouWantHere
```

This section explained the default implementation strategy for storing and checking permissions, using CustomData.  You can use this immediately, as it is the default behavior, and it should suit 95% of all use cases.

However, if you need another approach, you can fully customize how permissions are resolved for a given account or group by customizing the `ApplicationRealm`'s `accountPermissionResolver` and `groupPermissionResolver` properties, described next.

##### How Permission Checks Work #####

The Stormpath `ApplicationRealm` will use any configured `AccountPermissionResolver` and `GroupPermissionResolver` instances to create the aggregate of all permissions attributed to a `Subject` during a permission check.  In other words, the following call:

```java
subject.isPermitted(aPermission)
```

will return `true` if the following is true:

* any of the permissions returned by the `AccountPermissionResolver` for the Subject's backing Account implies `aPermission`
* any of the permissions returned by the `GroupPermissionResolver` for any of the backing Account's Groups implies `aPermission`

`false` will be returned if `aPermission` is not implied by any of these permissions.

For further clarity, the `isPermitted` check works something like this (simplified for brevity):

```java
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
```

###### AccountPermissionResolver

The StormpathRealm's `AccountPermissionResolver` inspects a Stormpath `Account` and returns a set of Shiro `Permission`s that are considered directly assigned to that `Account`.

This interface is provided to resolve permissions that are _directly_ assigned to a Stormpath `Account`.  Permissions that are assigned to an account's groups (and therefore implicitly or indirectly associated with an `Account`) are best provided by a `GroupPermissionResolver` instance instead.

Your `AccountPermissionResolver` implementation could then be configured on the StormpathRealm instance.  For example, in `shiro.ini`:

```ini
[main]
; ...
accountPermissionResolver = com.mycompany.stormpath.shiro.MyAccountPermissionResolver
stormpathRealm.accountPermissionResolver = $accountPermissionResolver
```

After you've configured this you can perform permission checks.  For example, perhaps you want to check if the current account is allowed to update their own information:

```java
String updateSelf = "account:" + subject.getPrincipal() + ":update";
if (subject.isPermitted(updateSelf)) {
    //do something
}
```

This check would succeed if the `MyAccountPermissionResolver` implementation returned that permission for the Subject's backing `Account`.

###### GroupPermissionResolver

The StormpathRealm's `GroupPermissionResolver` inspects a Stormpath `Group` and returns a set of Shiro `Permission`s that are considered assigned to that `Group`.

You can configure a custom `GroupPermissionResolver` implementation on the StormpathRealm instance.  For example, in `shiro.ini`:

```ini
[main]
; ...
groupPermissionResolver = com.mycompany.stormpath.shiro.MyGroupPermissionResolver
stormpathRealm.groupPermissionResolver = $groupPermissionResolver
```

After you've configured this you can perform group permission checks.  For example, perhaps you want to check if the current Subject is allowed to edit a specific blog article:

```java
String editArticle = "blogArticle:" + article.getId() + ":edit";
if (subject.isPermitted(editArticle)) {
    //do something
}
```

This check would succeed if the Subject's direct permissions _or any of its Groups' permissions_ (as returned by the `MyGroupPermissionResolver` implementation) implied the 'editArticle` permission.

## Caching

Reducing round-trips to the Stormpath API servers can be a beneficial performance boost, so you will likely want to enable caching.  

The <a href="wiki#configuration">Configuration</a> section example already shows caching being used, so if you copied that, you should be good to go - you don't need to configure anything additional.  However, if you are interested in what is going on, keep reading.

Shiro has its own Caching support that allows you to plug in to existing caching products (Hazelcast, Ehcache, etc).  The Stormpath Java SDK also has an identical concept for Stormpath users that don't rely on Shiro.

Instead of having to configure two caching mechanisms in your Shiro-enabled app (one for Shiro, and another separate one for the Stormpath Java SDK), the Apache Shiro plugin for Stormpath has a caching implementation that uses the Stormpath Java SDK caching API to 'wrap' or 'bridge' the Shiro caching API.  This means you can tell the Stormpath Java SDK to use the same `CacheManager` that Shiro uses, allowing you to use a single cache system for all of your application's needs.

To enable this 'bridge' support, for example, in `shiro.ini` (or the Spring, Guice or CDI equivalent):

```ini
[main]

; Enable whatever Shiro CacheManager implementation you want:
cacheManager = my.shiro.CacheManagerImplementation
securityManager.cacheManager = $cacheManager

; Stormpath integration:
stormpathClient = com.stormpath.shiro.client.ClientFactory
; Tell the stormpath client to use the same Shiro CacheManager:
stormpathClient.cacheManager = $cacheManager
```

If for some reason you _don't_ want the Stormpath SDK to use Shiro's caching mechanism, you can configure the `stormpathCacheManager` property (instead of the expected Shiro-specific `cacheManager` property), which accepts a `com.stormpath.sdk.cache.CacheManager` instance instead:

```ini
; ...
stormpathCacheManager = my.com.stormpath.sdk.cache.CacheManagerImplementation
; etc ...
stormpathClient.stormpathCacheManager = $stormpathCacheManager
```

But note this approach requires you to set-up/configure two separate caching mechanisms.

See `ClientFactory` `setCacheManager` and `setStormpathCacheManager` JavaDoc for more.