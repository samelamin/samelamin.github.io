---
layout: post
title: "Airflow AD Integration With Okta"
description: ""
category:
tags: [AD integration, OKTA, AIRFLOW]
---

#### I'm back!!!

After a very long hiatus I am finally back, so much has changed in my life that kept me away from the community. One of the prime reasons was I recently became a father of a beautiful baby girl! Now that the insomnia is gone I can get back to sharing what little knowledge I can with everyone. 

#### Why Airflow 
Loads of people use Airflow so I won't be going into why its a great workflow scheduler but I think one of the main problems in introducing it into established organizations (read corporate) is how hard it is to work with AD. AD is the defacto standard on how large corporations do authentication. Of course Airflow does support LDAP but that also means working with networking teams and a mountain of red tape that most engineers will just not see the ROI. OKTA is meant to solve this because it means I just have to talk to one team and as long as my application supports logging in via OKTA 


#### Ok then, it should be fairly straightforward...
Sadly there is a lack of resources on how to implement airflow with OKTA specifically. [This](https://support.okta.com/help/s/question/0D51Y00006JqJ4p/has-anyone-ever-integrated-okta-with-airflow-how-did-you-configure-the-airflow-backend?language=en_US) was posted in the OKTA forums recently but there isnt a clear guide on how to do it. Since Airflow is essentially a Flask app, we can integrate it via OKTA by adding some packages. This is what I will attempt to help with since I recently had to implement it for work

#### Enough talk, show me some code
If you are anything like me you are just scrolling through this post looking for code so that is why I try to have code as early as possible.  If possible, it's best to try this out locally, there are various docker-airflow setups you can use. I personally use Puckel's [repo](https://github.com/puckel/docker-airflow).

The first thing you need to do is create an OIDC app on your OKTA portal. You can easily set up your own developer portal to try this out! For this example let us take my portal of `"https://discoverysamelamin.okta.com` 

Once you do that you need 2 things, 
    - the client id and secret
    - The redirect URL. You will be using this to tell OKTA that this URL is a valid application

Once you have those 2 things, the hard part starts. First of all you need to enable `RBAC` in your `airflow.cfg` file. You will also need to set the environment variable `AIRFLOW__WEBSERVER__RBAC` to True'

This is needed by the packages to authenticate via OKTA
The packages needed can be found below

```
flask-oidc==1.4.0
fab-oidc==0.0.9
flask-bcrypt==0.7.1
```

Next, we use the client details to create the `client_secret.json`, this is the file that our FAB packages will use to know how to authenticate with OKTA. I saved this file on my AIRFLOW home directory `/usr/local/airflow/client_secret.json`. To help below is my file 

```

{
    "web": {
        "client_id": "xxxxxxxxx",
        "client_secret": "xxxxxxxxxxxxxx",
        "auth_uri": "https://discoverysamelamin.okta.com/oauth2/v1/authorize",
        "token_uri": "https://discoverysamelamin.okta.com/oauth2/v1/token",
        "userinfo_uri": "https://discoverysamelamin.okta.com/oauth2/v1/userinfo",
        "issuer": "https://discoverysamelamin.okta.com",
        "redirect_uris": [
            "http://localhost/*"
        ]
    }
}
```

We also need to generate a `webserver_config.py` file. I saved this in my Airflow HOME directory as well.

Please note that we have to pass the location of the `OIDC_CLIENT_SERCRETS` Mine was

```
# -*- coding: utf-8 -*-

"""Default configuration for the Airflow webserver"""
import os
from fab_oidc.security import AirflowOIDCSecurityManager
from flask_appbuilder.security.manager import AUTH_OID
from airflow.configuration import conf
SQLALCHEMY_DATABASE_URI = conf.get('core', 'SQL_ALCHEMY_CONN')
AUTH_ROLE_ADMIN = 'Admin'
AUTH_USER_REGISTRATION_ROLE = "Public"
AUTH_USER_REGISTRATION = True
basedir = os.path.abspath(os.path.dirname(__file__))
SECURITY_MANAGER_CLASS = AirflowOIDCSecurityManager
OIDC_CLIENT_SECRETS = '/usr/local/airflow/client_secret.json'
OIDC_SCOPES = ['openid', 'email', 'profile', 'groups']
AUTH_TYPE = AUTH_OID
OIDC_COOKIE_SECURE=False
OIDC_USER_INFO_ENABLED=True

```


Unfortunately, this isn't enough, we still need to patch on of the packages to make it work with Airflow. I generated a patch file which I then overwrite in the package directory.  The patch file is below

```
# Patch for fab-oidc to enable OKTA integration with Airflow
import os
import os
from flask import redirect, request
from flask_appbuilder.security.views import AuthOIDView
from flask_login import login_user
from flask_admin import expose
from urllib.parse import quote


class AuthOIDCView(AuthOIDView):

    @expose('/login/', methods=['GET', 'POST'])
    def login(self, flag=True):

        sm = self.appbuilder.sm
        oidc = sm.oid

        @self.appbuilder.sm.oid.require_login
        def handle_login():
            info = oidc.user_getinfo(['email', 'given_name', 'family_name'])
            email = info.get('email')
            user =  sm.find_user(username=email)
            if user is None:
                user = sm.add_user(
                    username=email,
                    first_name=info.get('given_name',email),
                    last_name=info.get('family_name',email),
                    email=email,
                    role=sm.find_role(sm.auth_user_registration_role)
                )
            
            login_user(user, remember=False)
            return redirect(self.appbuilder.get_url_for_index)

        return handle_login()

    @expose('/logout/', methods=['GET', 'POST'])
    def logout(self):

        oidc = self.appbuilder.sm.oid

        oidc.logout()
        super(AuthOIDCView, self).logout()
        redirect_url = request.url_root.strip(
            '/') + self.appbuilder.get_url_for_login

        logout_uri = oidc.client_secrets.get(
            'issuer') + '/protocol/openid-connect/logout?redirect_uri='
        if 'OIDC_LOGOUT_URI' in self.appbuilder.app.config:
            logout_uri = self.appbuilder.app.config['OIDC_LOGOUT_URI']

        return redirect(logout_uri + quote(redirect_url))
```

And you guessed it, I put this file in the Airflow directory. Also as part of my entrypoint script, I apply this patch. Below is a snippet from the entrypoint

```
    cp /usr/local/airflow/fab-oidc-patch.py /usr/local/airflow/.local/lib/python3.7/site-packages/fab_oidc/views.py
    rm -rf /usr/local/airflow/.local/lib/python3.7/site-packages/fab_oidc/__pycache__/
```

Please note my packages were installed on python3.7 so you might want to look into your environment and change accordingly 

### Final Gotchas
Before you enable login, you might need an admin user already set up on airflow, but I am yet to validate if this is needed or not so that is why I did not include it in this tutorial. But feel free to drop a comment or even better send a PR if that is the case or if you want to add any helpful comment for anyone attempting this

### Thank you! 
Finally thank you, I am not sure if people benefit from my data pipeline series but if you have, please drop me a note and ill add to it. Otherwise thank you for your time and as always feedback is always welcome