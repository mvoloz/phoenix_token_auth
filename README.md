PhoenixTokenAuth
================

Adds token authentication to Phoenix apps using Ecto.

An example app is available at https://github.com/manukall/phoenix_token_auth_react.

## Setup
You need to have a user model with at least the following schema:

```elixir
defmodule MyApp.User do
  use Ecto.Model

  schema "users" do
    field  :email,                       :string
    field  :hashed_password,             :string
    field  :hashed_confirmation_token,   :string
    field  :confirmed_at,                Ecto.DateTime
    field  :hashed_password_reset_token, :string
  end
end
```

Then add PhoenixTokenAuth to your Phoenix router:

```elixir
defmodule MyApp.Router do
  use Phoenix.Router
  require PhoenixTokenAuth

  pipeline :authenticated do
    plug PhoenixTokenAuth.Plug
  end

  scope "/api" do
    pipe_through :api

    PhoenixTokenAuth.mount
  end

  scope "/api" do
    pipe_through :authenticated
    pipe_through :api

    resources: messages, MessagesController
  end
end
```
This generates routes for sign-up and login and protects the messages resources from unauthenticated access.

The generated routes are:

method | path | description
-------|------|------------
POST | /api/users | sign up
POST | /api/users/:id/confirm | confirm account
POST | /api/session | login, will return a token as JSON
POST | /api/password_resets | request a reset-password-email
POST | /api/password_resets/reset | reset a password

Inside the controller, the authenticated user's id is accessible inside the connections assigns:

```elixir
def index(conn, _params) do
  user_id = conn.assigns.authenticated_user.id
  ...
end
```

Now add configuration:
```elixir
# config/config.exs
config :phoenix_token_auth,
  user_model: Myapp.User,                                                    # ecto model used for authentication
  repo: Myapp.Repo,                                                          # ecto repo
  crypto_provider: Comeonin.Bcrypt,                                          # crypto provider for hashing passwords/tokens. see http://hexdocs.pm/comeonin/
  token_secret: "the_very_secret_token",                                     # secret string used to sign the authentication token
  token_validity_in_minutes: 7 * 24 * 60                                     # minutes from login until a token expires
  email_sender: "myapp@example.com",                                         # sender address of emails sent by the app
  welcome_email_subject: fn user -> "Hello #{user.email}" end,               # function returning the subject of a welcome email
  welcome_email_body: fn user, confirmation_token -> confirmation_token end, # function returning the body of a welcome email
  password_reset_email_subject: fn user -> "Hello #{user.email}" end,        # function returning the subject of a welcome email
  password_reset_email_body: fn user, reset_token -> reset_token end,        # function returning the body of a welcome email
  mailgun_domain: "example.com"                                              # domain of your mailgun account
  mailgun_key: "secret"                                                      # secret key of your mailgun account
```


## Usage

### Signing up / Registering a new user
* POST request to /api/users.
* Body should be JSON encoded `{user: {email: "user@example.com", password: "secret"}}`.
* This will send an email containing the confirmation token.

### Confirming a user
* POST request to /api/users/:id/confirm
* Body should be JSON encoded `{confirmation_token: "token form the email"}`
* This will mark the user as confirmed and return an authentication token as JSON: `{token: "the_token"}`.

### Logging in
* POST request to /api/sessions
* Body should be JSON encoded `{email: "user@example.com", password: "secret"}`
* Will return an authentication token as JSON: `{token: "the_token"}`

### Requesting a protected resource
* Add a header with key `Authorization` and value `Bearer #{token}` to the request.
* `#{token}` is the token from either account confirmation or logging in.

### Logging out
* Logging out is completely client side. Just stop sending the `Authorization` header.

### Resetting password
* POST request to /api/password_resets
* Body should be JSON encoded `{email: "user@example.com"}`
* This will send an email as configured.
* Once the reset token is received in the email, make a POST request to /api/password_resets/reset with body
`{user_id: 123, password_reset_token: "the_token_from_the_email", password: "the_new_password"}`
* This will change the users password and return an authentication token as JSON: `{token: "the_token"}`.






## TODO:
* Better documentation
* Validations of email and password (format, length, ...)
