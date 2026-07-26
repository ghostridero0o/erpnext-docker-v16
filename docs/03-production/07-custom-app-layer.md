---
title: Lightweight Custom App Image
---

# Lightweight Custom App Image

This repository separates the ERPNext stack into two image layers:

- `erpnext-core`: Frappe, ERPNext, HRMS, CRM, Raven, Insights, Gameplan,
  Wiki, Print Designer, and Employee Self Service.
- `erpnext-custom`: `my_custom_app`, `frappe_crm_messenger`, and `furnicost`.

The core image is rebuilt only when a core app or its version changes. Normal
custom app deployments rebuild the small custom layer and do not run
`bench init` or Yarn for Raven.

## Build the core image

This is the slower build and should be run infrequently, preferably in CI or on
a dedicated build machine:

```console
docker build \
  --build-arg FRAPPE_BRANCH=version-16 \
  --build-arg CACHE_BUST=core-001 \
  --secret id=apps_json,src=apps-core.json \
  --tag erpnext-core:16-ready \
  --file images/layered/Containerfile .
```

## Build only the custom app layer

Use a cache-bust value made from the custom app commit IDs. This forces Docker
to fetch the current branch heads while leaving the core image untouched:

```console
docker build \
  --build-arg BASE_IMAGE=erpnext-core:16-ready \
  --build-arg CUSTOM_APPS_CACHE_BUST=a715f1c-a3b6d51-b2c41cb \
  --secret id=apps_json,src=apps-custom.json \
  --tag erpnext-custom:16-ready \
  --file images/custom-apps/Containerfile .
```

The resulting image records the exact source commit for each app at:

```text
/opt/frappe/custom-apps-revisions.txt
```

## Bootstrap from the existing all-in-one image

The first lightweight deployment can use the currently running all-in-one
image as its base. Give the result a new tag so the running image is unchanged
until the build succeeds:

```console
docker build \
  --build-arg BASE_IMAGE=erpnext-custom:16-staging-ready \
  --build-arg CUSTOM_APPS_CACHE_BUST=a715f1c-a3b6d51-b2c41cb \
  --secret id=apps_json,src=apps-custom.json \
  --tag erpnext-custom:16-custom-003 \
  --file images/custom-apps/Containerfile .
```

After validating the new image, point Compose at its tag, migrate the sites,
and recreate the application services. Database and Redis images are not part
of this custom layer.

## When a full core build is required

Rebuild `erpnext-core` when any of these change:

- Frappe, ERPNext, HRMS, CRM, Raven, Insights, Gameplan, Wiki, Print Designer,
  or Employee Self Service versions.
- Python, Node, operating-system packages, or the core Containerfile.
- A custom app adds a Python dependency that requires system build packages.
- A custom app introduces a JavaScript bundle requiring Node/Yarn compilation.

Python code, hooks, DocTypes, patches, and static public files in the three
custom apps only require rebuilding the custom layer followed by `bench migrate`.
