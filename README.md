# Table of Contents

- [Keycloak localization](#keycloak-localization)
- [Keycloak emails](#keycloak-emails)

## Keycloak localization

### Prerequisites

- Docker

### Description

This doc talks about how we can run `keycloak` project in a docker container and add your custom themes to extend existing languages for localization, change stylesheets for existing keycloak pages etc. First thing that we need to understand is that, we will be copying our `custom-theme` in the `themes` directory of existing `keycloak` image using `Dockerfile`. The name of the directory becomes the name of the theme, `custom-theme` in our case.

### Procedure

1. To create a theme called `custom-theme`, create the directory `themes/custom-theme`. Inside the theme directory, create a directory for each of the types your theme is going to provide.
   For example, to add the login type to the custom-theme theme, create the directory `themes/custom-theme/login`.
   For each type create a file `theme.properties` which allows setting some configuration for the theme.
   For example, to configure the theme `themes/custom-theme/login` to extend the base theme and import some common resources, create the file `themes/mytheme/login/theme.properties` with following contents:

```
    parent=base
    import=common/keycloak
```

2. Create the file `<THEME TYPE>/messages/messages\_<LOCALE>.properties` in the directory of your theme.

3. Add this file to the locales property in <THEME TYPE>/theme.properties. For a language to be available to users the realms login, account and email, the theme has to support the language, so you need to add your language for those theme types.
   For example, to add Nepali translations to the `custom-theme` theme create the file `themes/custom-theme/login/messages/messages_np.properties` with the following content. If you omit a translation for messages, they will use English.

```
    usernameOrEmail=इमेल
    password=पस्स्वोर्ड
```

4. Edit themes/mytheme/login/theme.properties and add:

```
    locales=en,np
```

5. Add the same for the `account` and `email` theme types. To do this create `themes/custom-theme/account/messages/messages_no.properties` and `themes/custom-theme/email/messages/messages_no.properties`. Leaving these files empty will result in the English messages being used.

6. Copy `themes/custom-theme/login/theme.properties` to `themes/custom-theme/account/theme.properties` and `themes/custom-theme/email/theme.properties`.

7. Add a translation for the language selector. This is done by adding a message to the English translation. To do this add the following to `themes/custom-theme/account/messages/messages_en.properties` and `themes/custom-theme/login/messages/messages_en.properties`:

```
    locale_np=नेपाली
```

8. Create a Dockerfile that extends existing keycloak project's image. There's a step that copies `custom-theme` to the working directory of keycloak

```dockerfile
FROM quay.io/keycloak/keycloak:latest as builder

# Enable health and metrics support
ENV KC_HEALTH_ENABLED=true
ENV KC_METRICS_ENABLED=true

# Configure a database vendor
ENV KC_DB=postgres

WORKDIR /opt/keycloak
# for demonstration purposes only, please make sure to use proper certificates in production instead
RUN keytool -genkeypair -storepass password -storetype PKCS12 -keyalg RSA -keysize 2048 -dname "CN=server" -alias server -ext "SAN:c=DNS:localhost,IP:127.0.0.1" -keystore conf/server.keystore
RUN /opt/keycloak/bin/kc.sh build

FROM quay.io/keycloak/keycloak:latest
COPY --from=builder /opt/keycloak/ /opt/keycloak/
COPY custom-theme /opt/keycloak/themes/custom-theme

ENTRYPOINT ["/opt/keycloak/bin/kc.sh"]
```

9. Use this Dockerfile in `docker-compose.yml` with a service called keycloak like

```yaml
keycloak:
  build:
    context: .
    dockerfile: Dockerfile
  environment:
    DB_VENDOR: postgres
    DB_ADDR: postgres
    DB_DATABASE: postgres
    DB_USER: postgres
    DB_PASSWORD: postgres
    KEYCLOAK_ADMIN: admin
    KEYCLOAK_ADMIN_PASSWORD: admin
  ports:
    - 7777:8080
  depends_on:
    - postgres
  command:
    - start-dev
    - --spi-theme-static-max-age=-1
    - --spi-theme-cache-themes=false
    - --spi-theme-cache-templates=false
```

10. Finally run `docker compose up --build` to start the keycloak server

### References

- [Keycloak base theme source code](https://github.com/keycloak/keycloak/tree/main/themes/src/main/resources/theme/base)
- [Adding a language to a realm](https://www.keycloak.org/docs/23.0.6/server_development/#adding-a-language-to-a-realm)
- [Adding a stylesheet to a theme](https://www.keycloak.org/docs/23.0.6/server_development/#add-a-stylesheet-to-a-theme)
- [Final result of above instructions](https://github.com/samya-ak/keycloak-localization/tree/main)

## Keycloak emails

### Description

> Keycloak sends emails to users
>
> - to verify their email address
> - when they forget their passwords
> - when an admin needs to receive notifications about a server event

We can setup SMTP in keycloak by following instructions from [here](https://wjw465150.gitbooks.io/keycloak-documentation/content/server_admin/topics/realms/email.html).

> Keycloak doesn’t send out mails immediately when a user is created. And also this depends if you have the email verification enabled. You can do this in two ways: In Realm Settings → Login → Verify Email or with the Required Action “Verify Email”.
> Both approaches will send an email to the user, if the user is not marked as “email verified” and is just trying to login. Required actions are fired at the end of an authentication flow and must be successfully solved, before.a user is completely authenticated.
> Keycloak has no thing like “invitation” which will be sent when the user has been created and tell the user “hey, you have just been created, please do… whatever you expect”
> But you can use the admin API right after creating a user to send a “credential reset” email with several required actions in it.
>
> - Mr. KeyCloak [here](https://keycloak.discourse.group/t/does-creating-a-new-user-via-api-send-a-confirmation-email/20646/4)

### Customizing email templates

To edit the subject and contents for emails, for example email verification email, add a message bundle to the email type of your theme. There are three messages for each email. One for the subject, one for the plain text body and one for the html body.

To see all emails available take a look at [here](https://github.com/keycloak/keycloak/blob/main/themes/src/main/resources/theme/base/email/messages/messages_en.properties).

For example to change the email verification email for the `custom-them` theme create `themes/custom-theme/email/messages/messages_en.properties` with the following content:

```
emailVerificationSubject=Verify email custom subject
emailVerificationBody=Someone has created a {2} account with this email address. If this was you, click the link below to verify your email address\n\n{0}\n\nThis link will expire within {3}.\n\nIf you didn''t create this account, just ignore this message.
emailVerificationBodyHtml=<p>Someone has created a {2} account with this email address. If this was you, click the link below to verify your email address</p><p><a href="{0}">Link to e-mail address verification</a></p><p>This link will expire within {3}.</p><p>If you didn''t create this account, just ignore this message lol.</p>
```

### Reference

[Emails docs](https://www.keycloak.org/docs/23.0.6/server_development/#emails)
