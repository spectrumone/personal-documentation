#Python Social Auth Guide
<i>So I dont have to rediscover them again</i>

###Create facebook app
* create an app here [https://developers.facebook.com/apps/](https://developers.facebook.com/apps/). The name of the app can be anything.
* on the redirect uri, you cant use <b>127.0.0.1:8000</b> so for testing it locally, use <b>localhost:8000</b> instead.

###settings.py
Most of these are not well documented in the site.
```python
SOCIAL_AUTH_USER_MODEL = 'django.contrib.auth.models.User' # used to point where the User model is. Needed for creating users.
SOCIAL_AUTH_FACEBOOK_KEY = ''  # from facebook app
SOCIAL_AUTH_FACEBOOK_SECRET = '' # from facebook app
SOCIAL_AUTH_FACEBOOK_SCOPE = ['public_profile', 'friends', 'email'] # scope of your access token https://developers.facebook.com/docs/facebook-login/permissions
SOCIAL_AUTH_FACEBOOK_PROFILE_EXTRA_PARAMS = {
    'fields': 'name, email', 
} # params to facebook api call using uid of the Facebook User

# pipelines to be run by psa
# can be overriden. Check code at source.
SOCIAL_AUTH_PIPELINE = (
    'social.pipeline.social_auth.social_details',
    'social.pipeline.social_auth.social_uid',
    'social.pipeline.social_auth.auth_allowed',
    'social.pipeline.social_auth.social_user',
    'social.pipeline.social_auth.associate_by_email',
    'social.pipeline.social_auth.associate_user',
    'social.pipeline.social_auth.load_extra_data',
    'social.pipeline.user.user_details',
)

INSTALLED_APPS = (
    ...
    'social.apps.django_app.default' # to include it in django path.

# where the template context processors are located depends on your django version   
TEMPLATE_CONTEXT_PROCESSORS = (
    'social.apps.django_app.context_processors.backends',
    'social.apps.django_app.context_processors.login_redirect',
)

# needed to run authentication backend of facebook.
# can be overriden by copying code from source. 
AUTHENTICATION_BACKENDS = [
    ...
    'social.backends.facebook.FacebookOAuth2'
]
```

###urls.py
This is needed register psa urls.
```python
urlpatterns = [
    ...
    url(r'^social/', include('social.apps.django_app.urls', namespace='social')),
```

###base.html
Href button for the social auth. After authorization, will redirect to request.path i think or the redirect uri specified on registering the facebook api.
```python
<a href="{% url 'social:begin' 'facebook' %}?next={{ request.path }}"> 
```

###Default Pipelines
Wrote this down along with their description so you dont have to look for them at source when you want to create 
```python
'social.pipeline.social_auth.social_details'
    # Get the information we can about the user and return it in a simple
    # format to create the user instance later. On some cases the details are
    # already part of the auth response from the provider, but sometimes this
    # could hit a provider API.
def social_details(backend, response, *args, **kwargs):
    return {'details': backend.get_user_details(response)}


'social.pipeline.social_auth.social_uid'
    # Get the social uid from whichever service we're authing thru. The uid is
    # the unique identifier of the given user in the provider.
def social_uid(backend, details, response, *args, **kwargs):
    return {'uid': backend.get_user_id(details, response)}


'social.pipeline.social_auth.auth_allowed'
    # Verifies that the current auth process is valid within the current
    # project, this is where emails and domains whitelists are applied (if
    # defined).
def auth_allowed(backend, details, response, *args, **kwargs):
    if not backend.auth_allowed(response, details):
        raise AuthForbidden(backend)


'social.pipeline.social_auth.social_user'
    # Checks if the current social-account is already associated in the site.
def social_user(backend, uid, user=None, *args, **kwargs):
    provider = backend.name
    social = backend.strategy.storage.user.get_social_auth(provider, uid)
    if social:
        if user and social.user != user:
            msg = 'This {0} account is already in use.'.format(provider)
            raise AuthAlreadyAssociated(backend, msg)
        elif not user:
            user = social.user
    return {'social': social,
            'user': user,
            'is_new': user is None,
            'new_association': False}


'social.pipeline.social_auth.associate_by_email',
    # Associates the current social details with another user account with
    # a similar email address. Disabled by default.
def associate_by_email(backend, details, user=None, *args, **kwargs):
    """
    Associate current auth with a user with the same email address in the DB.

    This pipeline entry is not 100% secure unless you know that the providers
    enabled enforce email verification on their side, otherwise a user can
    attempt to take over another user account by using the same (not validated)
    email address on some provider.  This pipeline entry is disabled by
    default.
    """
    if user:
        return None

    email = details.get('email')
    if email:
        # Try to associate accounts registered with the same email address,
        # only if it's a single object. AuthException is raised if multiple
        # objects are returned.
        users = list(backend.strategy.storage.user.get_users_by_email(email))
        if len(users) == 0:
            return None
        elif len(users) > 1:
            raise AuthException(
                backend,
                'The given email address is associated with another account'
            )
        else:
            return {'user': users[0]}


'social.pipeline.user.get_username',
    # Make up a username for this person, appends a random string at the end if
    # there's any collision.
def get_username(strategy, details, user=None, *args, **kwargs):
    if 'username' not in strategy.setting('USER_FIELDS', USER_FIELDS):
        return
    storage = strategy.storage

    if not user:
        email_as_username = strategy.setting('USERNAME_IS_FULL_EMAIL', False)
        uuid_length = strategy.setting('UUID_LENGTH', 16)
        max_length = storage.user.username_max_length()
        do_slugify = strategy.setting('SLUGIFY_USERNAMES', False)
        do_clean = strategy.setting('CLEAN_USERNAMES', True)

        if do_clean:
            clean_func = storage.user.clean_username
        else:
            clean_func = lambda val: val

        if do_slugify:
            override_slug = strategy.setting('SLUGIFY_FUNCTION')
            if override_slug:
                slug_func = module_member(override_slug)
            else:
                slug_func = slugify
        else:
            slug_func = lambda val: val

        if email_as_username and details.get('email'):
            username = details['email']
        elif details.get('username'):
            username = details['username']
        else:
            username = uuid4().hex

        short_username = username[:max_length - uuid_length]
        final_username = slug_func(clean_func(username[:max_length]))

        # Generate a unique username for current user using username
        # as base but adding a unique hash at the end. Original
        # username is cut to avoid any field max_length.
        while storage.user.user_exists(username=final_username):
            username = short_username + uuid4().hex[:uuid_length]
            final_username = slug_func(clean_func(username[:max_length]))
    else:
        final_username = storage.user.get_username(user)
    return {'username': final_username}


'social.pipeline.user.create_user'
    # Create a user account if we haven't found one yet.
def create_user(strategy, details, user=None, *args, **kwargs):
    if user:
        return {'is_new': False}
    fields = dict((name, kwargs.get(name) or details.get(name))
                        for name in strategy.setting('USER_FIELDS',
                                                      USER_FIELDS))
    if not fields:
        return

    return {
        'is_new': True,
        'user': strategy.create_user(**fields)
    }


'social.pipeline.social_auth.associate_user'
    # Create the record that associated the social account with this user.
def associate_user(backend, uid, user=None, social=None, *args, **kwargs):
    if user and not social:
        try:
            social = backend.strategy.storage.user.create_social_auth(
                user, uid, backend.name
            )
        except Exception as err:
            if not backend.strategy.storage.is_integrity_error(err):
                raise
            # Protect for possible race condition, those bastard with FTL
            # clicking capabilities, check issue #131:
            #   https://github.com/omab/django-social-auth/issues/131
            return social_user(backend, uid, user, *args, **kwargs)
        else:
            return {'social': social,
                    'user': social.user,
                    'new_association': True}


'social.pipeline.social_auth.load_extra_data'
    # Populate the extra_data field in the social record with the values
    # specified by settings (and the default ones like access_token, etc).
def load_extra_data(backend, details, response, uid, user, *args, **kwargs):
    social = kwargs.get('social') or \
             backend.strategy.storage.user.get_social_auth(backend.name, uid)
    if social:
        extra_data = backend.extra_data(user, uid, response, details)
        social.set_extra_data(extra_data)


'social.pipeline.user.user_details'
    # Update the user record with any changed info from the auth service.
def user_details(strategy, details, user=None, *args, **kwargs):
    """Update user details using data from provider."""
    if user:
        changed = False  # flag to track changes
        protected = strategy.setting('PROTECTED_USER_FIELDS', [])
        keep = ('username', 'id', 'pk') + tuple(protected)

        for name, value in details.items():
            # do not update username, it was already generated
            # do not update configured fields if user already existed
            if name not in keep and hasattr(user, name):
                if value and value != getattr(user, name, None):
                    try:
                        setattr(user, name, value)
                        changed = True
                    except AttributeError:
                        pass

        if changed:
            strategy.storage.user.changed(user)
```
