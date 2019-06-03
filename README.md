[![Build Status](https://travis-ci.org/magnetcoop/dashboard-manager.grafana.svg?branch=master)](https://travis-ci.org/magnetcoop/dashboard-manager.grafana)

# dashboard-manager.grafana

A [Duct](https://github.com/duct-framework/duct) library that provides an [Integrant](https://github.com/weavejester/integrant) key for managing dashboards and associated users and organizations in Grafana.

## Table of contents
* [Installation](#installation)
* [Usage](#usage)
  * [Configuration](#configuration)
  * [Managing organizations](#managing-organizations)
    * [create-org](#create-org)
    * [get-orgs](#get-orgs)
    * [update-org](#update-org)
    * [delete-org](#delete-org)
    * [add-org-user](#add-org-user)
    * [get-org-users](#get-org-users)
  * [Managing users](#managing-users)
    * [create-user](#create-user)
    * [update-user](#update-user)
    * [get-user](#get-user)
    * [get-user-orgs](#get-user-orgs)
  * [Managing dashboards](#managing-dashboards)
    * [get-org-dashboards](#get-org-dashboards)
    * [get-org-panels](#get-org-panels)
    * [get-ds-panels](#get-ds-panels)

## Installation

[![Clojars Project](https://clojars.org/magnet/dashboard-manager.grafana/latest-version.svg)](https://clojars.org/magnet/dashboard-manager.grafana)

## Usage

### Configuration
To use this library add the following key to your configuration:

`:magnet.dashboard-manager/grafana`

This key expects a configuration map with two mandatory keys, plus another three optional ones.
These are the mandatory keys:

* `:uri` : The URI where Grafana's server is listening.
* `:credentials`: A vector with two elements, a username and password, that are used for basic HTTP authentication.

These are the optional keys:
* `:timeout`: Timeout value (in milli-seconds) for an connection attempt with Grafana.
* `:max-retries`: If the connection attempt fails, how many retries we want to attempt before giving up.
* `:backoff-ms`: This is a vector in the form [initial-delay-ms max-delay-ms multiplier] to control the delay between each retry. The delay for nth retry will be (max (* initial-delay-ms n multiplier) max-delay-ms). If multiplier is not specified (or if it is nil), a multiplier of 2 is used. All times are in milli-seconds.

Key initialization returns a `Grafana` record that can be used to perform the Grafana operations described below.

#### Configuration example
Basic configuration:
```edn
  :magnet.dashboard-manager/grafana
   {:uri  #duct/env ["GRAFANA_URI" Str :or "http://localhost:3000"]
    :credentials [#duct/env ["GRAFANA_USERNAME" Str :or "admin"] 
                  #duct/env ["GRAFANA_TEST_PASSWORD" Str :or "admin"]]}
```

Configuration with custom request retry policy:
```edn
  :magnet.dashboard-manager/grafana
   {:uri #duct/env ["GRAFANA_URI" Str :or "http://localhost:3000"]
    :credentials [#duct/env ["GRAFANA_USERNAME" Str :or "admin"] 
                  #duct/env ["GRAFANA_TEST_PASSWORD" Str :or "admin"]]
    :timeout 300
    :max-retries 5
    :backoff-ms [10 500]}
```

### Obtaining a `Grafana` record

If you are using the library as part of a [Duct](https://github.com/duct-framework/duct)-based project, adding any of the previous configurations to your `config.edn` file will perform all the steps necessary to initialize the key and return a `Grafana` record for the associated configuration. In order to show a few interactive usages of the library, we will do all the steps manually in the REPL.

First we require the relevant namespaces:

```clj
user> (require '[integrant.core :as ig]
               '[magnet.dashboard-manager.core :as core])
nil
user>
```

Next we create the configuration var holding the Grafana integration configuration details:

```clj
user> (def config {:uri #duct/env ["GRAFANA_URI" Str :or "http://localhost:3000"]
                   :credentials [#duct/env ["GRAFANA_USERNAME" Str :or "admin"] 
                                 #duct/env ["GRAFANA_PASSWORD" Str :or "admin"]]})
#'user/config
user>
```

Now that we have all pieces in place, we can initialize the `:magnet.dashboard-manager/grafana` Integrant key to get a `Grafana` record. As we are doing all this from the REPL, we have to manually require `magnet.dashboard-manager.grafana` namespace, where the `init-key` multimethod for that key is defined (this is not needed when Duct takes care of initializing the key as part of the application start up):

``` clj
user> (require '[magnet.dashboard-manager.grafana :as grafana])
nil
user>
```

And we finally initialize the key with the configuration defined above, to get our `Grafana` record:

``` clj
user> (def gf-record (->
                       config
                       (->> (ig/init-key :magnet.dashboard-manager/grafana))))
#'user/gf-record
user> gf-record
#magnet.dashboard_manager.grafana.Grafana{:uri "http://localhost:4000",
                                          :credentials ["admin"
                                                        "admin"],
                                          :timeout 200,
                                          :max-retries 10,
                                          :backoff-ms [500 1000 2.0]}
user>
```
Now that we have our `Grafana` record, we are ready to use the methods defined by the protocols defined in `magnet.dashboard-manager.core` namespace.

### Managing organizations
#### `create-org`
* parameters: 
  - A `Grafana` record
  - Organization name
* returning value:
  - `:status`: `:ok`, `:access-denied`, `:not-found`, `:unknown-host`, `:connection-refused`, `:error`, `:already-exists`
  - `:orgId`: ID assigned to the created organization
* Example:
```clj
user> (core/create-org gf-record "foo")
{:status :ok :orgId 2}
```
#### `get-orgs`
* parameters: 
  - A `Grafana` record
* returning value:
  - `:status`: `:ok`, `:access-denied`, `:not-found`, `:unknown-host`, `:connection-refused`, `:error`
  - `:orgs`: A vector of maps. Each map representing an existing organization. 
* Example:
```clj
user> (core/get-orgs gf-record)
{:status :ok :orgs [{:id 1 :name "Main Org"}
                    {:id 2 :name "foo"}]}
```
#### `update-org`
* parameters: 
  - A `Grafana` record
  - Organization ID
  - Organization's new name
* returning value:
  - `:status`: `:ok`, `:access-denied`, `:not-found`, `:unknown-host`, `:connection-refused`, `:error`, `:already-exists`
* Example:
```clj
user> (core/update-org gf-record 2 "foo-bar")
{:status :ok}
```
#### `delete-org`
* parameters: 
  - A `Grafana` record
  - Organization ID
* returning value:
  - `:status`: `:ok`, `:access-denied`, `:not-found`, `:unknown-host`, `:connection-refused`, `:error`
* Example:
```clj
user> (core/delete-org gf-record 2)
{:status :ok}
```
#### `add-org-user`
* description: Adds a user to an organization with a specific role.
* parameters: 
  - A `Grafana` record
  - Organization ID
  - User's login name (username or email)
  - User's role: Viewer, Editor or Admin
* returning value:
  - `:status`: `:ok`,`:access-denied`, `:not-found`, `:error`,`:role-not-found`, `:user-not-found`, `:already-exists`
* Example:
```clj
user> (core/add-org-user gf-record 1 "foo-bar" "Editor")
{:status :ok}
```
#### `get-org-users`
* description: Gets the user list for the given organization.
* parameters: 
  - A `Grafana` record
  - Organization ID
* returning value:
  - `:status`: `:ok`,`:access-denied`, `:not-found`, `:error`, `:not-found`
  - `:users`: A vector of maps. Each map representing an existing user.
* Example:
```clj
user> (core/get-org-users gf-record 1)
{:status :ok :users [{:orgId 1, :userId 1, :email "admin@localhost", :avatarUrl "/avatar/46d229b033af06a191ff2267bca9ae56", :login "admin", :role "Admin", :lastSeenAt "2019-05-27T14:21:51Z", :lastSeenAtAge "< 1m"}
                     {:orgId 1, :userId 2, :email "foo-bar@email.com", :avatarUrl "/avatar/46d234t033af06a191ff2267bca9ae56", :login "foo-bar", :role "Editor", :lastSeenAt "2019-05-27T14:21:51Z", :lastSeenAtAge "< 1m"}]}
```
### Managing users
#### `create-user`
* parameters: 
  - A `Grafana` record
  - User data: a map with the user specific data
    - `:name` (OPTIONAL)
    - `:email` (REQUIRED if login is not specified)
    - `:login` (REQUIRED if email is not specified)
    - `:password` (REQUIRED)
* returning value:
  - `:status`: `:ok`, `:access-denied`, `:not-found`, `:unknown-host`, `:connection-refused`, `:error`, `:already-exists`, `:invalid-data`
  -  `:id`: The created user's ID.
* Example:
```clj
user> (core/create-user gf-record {:login "login" :password "password"})
{:status :ok :id 3}
```
#### `update-user`
* parameters: 
  - A `Grafana` record
  - User ID
  - User data: a map with the data we want to change, plus the login field
    - `:name` 
    - `:email` (REQUIRED if login is not specified)
    - `:login` (REQUIRED if email is not specified)
* returning value:
  - `:status`: `:ok`, `:access-denied`, `:not-found`, `:unknown-host`, `:connection-refused`, `:error`, `:already-exists`, `:missing-mandatory-data`
  -  `:id`: The created user's ID.
* Example:
```clj
user> (core/update-user gf-record 3 {:name "fooo" :login "login"})
{:status :ok}
```
#### `get-user`
* parameters: 
  - A `Grafana` record
  - User's login or email
* returning value:
  - `:status`: `:ok`, `:access-denied`, `:not-found`, `:unknown-host`, `:connection-refused`, `:error`
  - `:user`: A map with the user information.
* Example:
```clj
user> (core/get-user gf-record "login")
{:status :ok :user {:id 3, :email "fooo@email.com", :name "fooo", :login "login", :theme "", :orgId 1, :isGrafanaAdmin false}}
```
#### `get-user-orgs`
* description: Gets a list of organizations to which a user belongs.
* parameters: 
  - A `Grafana` record
  - User ID
* returning value:
  - `:status`: `:ok`, `:access-denied`, `:not-found`, `:unknown-host`, `:connection-refused`, `:error`, `not-found`
  - `:orgs`: A vector of maps. Each map representing an organization.
* Example:
```clj
user> (core/get-user-orgs gf-record 1)
{:status :ok :orgs [{:orgId 1, :name "Main Org.", :role "Admin"}]}
```
### Managing dashboards
#### `get-org-dashboards`
* description: Gets a list of dashboards for the given organization.
* parameters: 
  - A `Grafana` record
  - Organization ID
* returning value:
  - `:status`: `:ok`, `:access-denied`, `:not-found`, `:unknown-host`, `:connection-refused`, `:error`
  - `:dashboards`: A list of maps. Each map representing a dashboard and it's panels.
* Example:
```clj
user> (core/get-org-dashboards gf-record 1)
{:status :ok, :dashboards ({:uid "yYtEB6WZz", :title "Example Dashboard", :url "/d/yYtEB6WZz/example-dashboard", :panels ({:id 2, :title "Panel Title\
", :ds-url "/d/yYtEB6WZz/example-dashboard"} {:id 4, :title "Panel Title", :ds-url "/d/yYtEB6WZz/example-dashboard"})})}
```
#### `get-org-panels`
* description: Gets a list of panels for the given organization.
* parameters: 
  - A `Grafana` record
  - Organization ID
* returning value:
  - `:status`: `:ok`, `:access-denied`, `:not-found`, `:unknown-host`, `:connection-refused`, `:error`
  - `:panels`: A list of maps. Each map representing a panel.
* Example:
```clj
user> (core/get-org-panels gf-record 1)
{:status :ok, :panels ({:id 2, :title "Panel Title", :ds-url "/d/yYtEB6WZz/example-dashboard"} {:id 4, :title "Panel Title", :ds-url "/d/yYtEB6WZz/ex\
ample-dashboard"})}
```
#### `get-ds-panels`
* description: Gets a list of panels for the given dashboard.
* parameters: 
  - A `Grafana` record
  - Organization ID
  - Dashboard ID
* returning value:
  - `:status`: `:ok`, `:access-denied`, `:not-found`, `:unknown-host`, `:connection-refused`, `:error`
  - `:panels`: A list of maps. Each map representing a panel.
* Example:
```clj
user> (core/get-org-panels gf-record 1)
{:status :ok, :panels ({:id 2, :title "Panel Title", :ds-url "/d/yYtEB6WZz/example-dashboard"} {:id 4, :title "Panel Title", :ds-url "/d/yYtEB6WZz/ex\
ample-dashboard"})}
```
## License

Copyright (c) Magnet S Coop 2019.

The source code for the library is subject to the terms of the Mozilla Public License, v. 2.0. If a copy of the MPL was not distributed with this file, You can obtain one at https://mozilla.org/MPL/2.0/.
