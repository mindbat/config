# Config


A library for declaring configuration vars in a centralized fashion. Included tooling allows one to gather and emit all configured vars and their docstrings, default values, etc.


## Motivation

Most configuration systems rely on consumers pulling named values from something akin to a global map (cf. [environ](https://github.com/weavejester/environ)). This yields a number of negative consequences:

- It is difficult to identify what configuration values are required by a system.
- There is no single place to put documentation for a configurable entry.
- Multiple consumers may independently pull directly from the configuration source, leaving only the configuration key to reveal their shared dependency.
- Default values for missing configuration can be inconsistent across consumers.
- Sources of configuration values are widely varied, e.g., properties files, system environment variables.

This library attempts to address the above issues by:

- Using declaration of configurable vars, which can be documented, defaulted, and used as a canonical source of data for other code just as any other library-specific var would be.
- Providing introspection into the configurable surface of an application and its dependencies.
- Relying on pushing values to all configuration vars, and doing so from a single source.


## Overview

Configuration is provided in an [EDN](http://edn-format.org) map of namespaced symbols (naming a config var) to the value to be bound to the corresponding var:

```clojure
{
com.example/greeting       "Hello World!"
com.example/tree           {:id 1, :children #{{:id 2} {:id 3}}}
com.example/aws-secret-key #config/env "AWS_SECRET_KEY"
}
```

As shown above, a custom data-reader (`#config/env`) has been provided to allow for pulling in values from the environment. If the environment does not have that entry, the var will use its default value or remain unbound.

The configuration EDN map is provided to an application in one of two ways:

1. A `config.edn` file in the current working directory.
2. A `config.edn` java system property (e.g., a command line arg `-Dconfig.edn=...`). The value can be any string consumable by [`clojure.java.io/reader`](http://clojure.github.io/clojure/clojure.java.io-api.html#clojure.java.io/reader).

If both are provided, the system property will be used.

Provisioning via environment variable is intentionally unsupported, though feel free to use something like `-Dconfig.edn=$CONFIG_EDN` when starting an application.


## Installation

Applications and libraries wishing to declare config vars, add the following dependency in your `project.clj` file:

```clojure
:dependencies [[outpace/config "0.1.0"]]
```

Note: it is inappropriate for libraries to include their own `config.edn` file since that is an application deployment concern. Including default values in-code (which can then be exposed by the generator) is acceptable.


## Config Usage

Declaring config vars is straightforward:

```clojure
(require '[outpace.config :refer [defconfig]])

(defconfig my-var)

(defconfig var-with-default 42)

(defconfig ^:dynamic *rebindable-var*)

(defconfig! required-var)
```

As shown above, the `defconfig` form supports anything a regular `def` form does. Additionally, for config vars without a default value you can use `defconfig!` to throw an error if no configured value is provided.

The `outpace.config` namespace includes the current state of the configuration, and while it can be used by code to explicitly pull config values, **this is strongly discouraged**; just use `defconfig`.

## Generator Usage

The `outpace.config.generate` namespace exists to generate a `config.edn` file containing everything one may need to know about the state of the config vars in the application and its dependent namespaces. If a `config.edn` file is already present, its contents will be loaded, and thus preserved by the replacing file.

To generate a `config.edn` file, invoke the following the the same directory as your `project.clj` file:

```bash
lein run -m outpace.config.generate
```

Alternately, one can just invoke `lein config` by adding the following to `project.clj`:

```clojure
:aliases {"config" ["run" "-m" "outpace.config.generate"]}
```

The generator can take an optional `:strict` flag (e.g., `lein config :strict`) that will result in an exception after file generation if there are any config vars with neither a default value nor configured value.

The following is an example of a generated `config.edn` file:

```clojure
{

;; UNBOUND CONFIG VARS:

; This is the docstring for the 'foo' var. This
; var does not have a default value.
#_com.example/foo


;; UNUSED CONFIG ENTRIES:

com.example/bar 123


;; CONFIG ENTRIES:

; The docstring for aaa.
com.example/aaa :configured-aaa

; The docstring for bbb. This var has a default value.
com.example/bbb :configured-bbb #_:default-bbb

; The docstring for ccc. This var has a default value.
#_com.example/ccc #_:default-ccc

}
```

The first section lists commented-out config vars that do not have a default value nor configured value, thus will be unbound at runtime. If a config value is provided, these entries will be re-categorized after regeneration.

The second section lists config entries that have no corresponding config var. This may happen after code change, or when a dependent library has been removed. If the config var reappears, these entries will be re-categorized after regeneration.

The third section lists all config vars used by the system, and their respective values.  For reference purposes, commented-out default values will be included after the configured value.  Likewise, commented-out entries will be included when their default values are used.
