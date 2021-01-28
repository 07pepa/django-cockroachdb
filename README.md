# CockroachDB backend for Django

## Prerequisites

You must install either:

* [psycopg2](https://pypi.org/project/psycopg2/), which has some
  [prerequisites](https://www.psycopg.org/docs/install.html#prerequisites) of
  its own.

* [psycopg2-binary](https://pypi.org/project/psycopg2-binary/)

The binary package is a practical choice for development and testing but in
production it is advised to use the package built from sources.

## Install and usage

Use the version of django-cockroachdb that corresponds to your version of
Django. For example, to get the latest compatible release for Django 3.1.x:

`pip install django-cockroachdb==3.1.*`

The minor release number of Django doesn't correspond to the minor release
number of django-cockroachdb. Use the latest minor release of each.

Configure the Django `DATABASES` setting similar to this:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django_cockroachdb',
        'NAME': 'django',
        'USER': 'myprojectuser',
        'PASSWORD': '',
        'HOST': 'localhost',
        'PORT': '26257',
        # If connecting with SSL, include the section below, replacing the
        # file paths as appropriate.
        'OPTIONS': {
            'sslmode': 'verify-full',
            'sslrootcert': '/certs/ca.crt',
            # Either sslcert and sslkey (below) or PASSWORD (above) is
            # required.
            'sslcert': '/certs/client.myprojectuser.crt',
            'sslkey': '/certs/client.myprojectuser.key',
        },
    },
}
```

If using Kerberos authentication, you can specify a custom service name in
`'OPTIONS'` using the key `'krbsrvname'`.

## Notes on Django fields

- `IntegerField` uses the same storage as `BigIntegerField` so `IntegerField`
  is introspected by `inspectdb` as `BigIntegerField`.

- `AutoField` and `BigAutoField` are both stored as
  [integer](https://www.cockroachlabs.com/docs/stable/int.html) (64-bit) with
  [`DEFAULT unique_rowid()`](https://www.cockroachlabs.com/docs/stable/functions-and-operators.html#id-generation-functions).

## Notes on Django QuerySets

- [`QuerySet.explain()`](https://docs.djangoproject.com/en/stable/ref/models/querysets/#explain)
  accepts `verbose`, `types`, `opt`, `vec`, and `distsql` options which
  correspond to [CockroachDB's parameters](https://www.cockroachlabs.com/docs/stable/explain.html#parameters).
  For example:

   ```python
   >>> Choice.objects.explain(opt=True, verbose=True)
   'scan polls_choice\n ├── columns: id:1 question_id:4 choice_text:2 votes:3\n ├── stats: [rows=1]\n ├── cost: 1.1\n ├── key: (1)\n ├── fd: (1)-->(2-4)\n └── prune: (1-4)'
   ```

## FAQ

### Why do I get the error ``psycopg2.errors.InvalidName: no database specified``?

You may need to [create the database](https://www.cockroachlabs.com/docs/stable/create-database.html).
You can use `cockroach sql --insecure` on the command line to get a SQL prompt.

## GIS support

Starting with CockroachDB 20.2.x and django-cockroachdb 3.1.3, you can use
`django.contrib.gis` with CockroachDB.

Use `'ENGINE': 'django_cockroachdb_gis'` in Django's `DATABASES` setting.

You must [install GEOS](https://docs.djangoproject.com/en/stable/ref/contrib/gis/install/geolibs/)
rather than have Django use the `libgeos_c.so` bundled with CockroachDB. In
other words, if you try this Django setting:

`GEOS_LIBRARY_PATH = '/usr/local/lib/cockroach/libgeos_c.so'`

It will fail with `OSError: libgeos.so.3.8.1: cannot open shared object file:
No such file or directory`.

## Known issues and limitations (as of CockroachDB 20.2.6)

- CockroachDB [can't disable constraint checking](https://github.com/cockroachdb/cockroach/issues/19444),
  which means certain things in Django like forward references in fixtures
  aren't supported.

- Migrations have some limitations. CockroachDB doesn't support:

   - [changing column type](https://github.com/cockroachdb/cockroach/issues/9851)
   - dropping or changing a table's primary key

- Known Bugs:
   - [Timezones after 2038 use incorrect DST settings](https://github.com/cockroachdb/django-cockroachdb/issues/124).

- Unsupported queries:
   - [Mixed type addition in SELECT](https://github.com/cockroachdb/django-cockroachdb/issues/19):
     `unsupported binary operator: <int> + <float>`
   - [UPDATE float column with integer column](https://github.com/cockroachdb/django-cockroachdb/issues/20):
     `value type int doesn't match type FLOAT8 of column <name>`
   - [Division that yields a different type](https://github.com/cockroachdb/django-cockroachdb/issues/21):
     `unsupported binary operator: <int> / <int> (desired <int>)`
   - [The power() database function doesn't accept negative exponents](https://github.com/cockroachdb/django-cockroachdb/issues/22):
     `power(): integer out of range`
   - [sum() doesn't support arguments of different types](https://github.com/cockroachdb/django-cockroachdb/issues/73):
      `sum(): unsupported binary operator: <float> + <int>`
   - [greatest() doesn't support arguments of different types](https://github.com/cockroachdb/django-cockroachdb/issues/74):
     `greatest(): expected <arg> to be of type <type>, found type <other type>`
   - [`SmallAutoField` generates values that are too large for any corresponding foreign keys](https://github.com/cockroachdb/django-cockroachdb/issues/84).
   - [The `SHA224` and `SHA384` database functions aren't supported](https://github.com/cockroachdb/django-cockroachdb/issues/81).

- GIS:
   - Some database functions aren't supported: `AsGML`, `AsKML`, `AsSVG`,
     `BoundingCircle`,  `GeometryDistance`, `LineLocatePoint`, and `MemSize`.
   - The `Length` database function isn't supported on geodetic fields:
     [st_lengthspheroid(): unimplemented](https://github.com/cockroachdb/cockroach/issues/48968).
   - `Union` may crash with
     [unknown signature: st_union(geometry, geometry)](https://github.com/cockroachdb/cockroach/issues/49064).
   - The spheroid argument of ST_DistanceSpheroid
     [isn't supported](https://github.com/cockroachdb/cockroach/issues/48922):
     `unknown signature: st_distancespheroid(geometry, geometry, string)`.
   - 3D storage isn't supported.
   - These lookups aren't supported:
     - [contained (@)](https://github.com/cockroachdb/cockroach/issues/56124)
     - [exact/same_as (~=)](https://github.com/cockroachdb/cockroach/issues/57096)
     - [left (<<) and right (>>)](https://github.com/cockroachdb/cockroach/issues/57092)
     - [overlaps_left (&<), overlaps_right (&>), overlaps_above (&<|),
       overlaps_below (&>|)](https://github.com/cockroachdb/cockroach/issues/57098)
     - [strictly_above (|>>), strictly_below (<<|)](https://github.com/cockroachdb/cockroach/issues/57095)
