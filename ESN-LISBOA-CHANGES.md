# ESN Lisboa deployment changes

This fork of [uatisdeproblem/esn-assembly](https://github.com/uatisdeproblem/esn-assembly) is deployed for ESN Lisboa at `https://ga.esnlisboa.org`.
This document lists what differs from the original project and the manual infrastructure setup it relies on.

## Environment

| Item | Value |
|---|---|
| AWS account | `085603761600` |
| AWS CLI profile | `esnlisboa` |
| Project key (CDK stack prefix) | `esn-assembly-lisboa-2` |
| Back-end region | `eu-west-1` (Ireland) |
| Email (SES) region | `eu-central-1` (Frankfurt) |
| Front-end | `ga.esnlisboa.org` |
| API | `api.ga.esnlisboa.org` |
| WebSocket API | `socket.ga.esnlisboa.org` |
| Media | `media.ga.esnlisboa.org` |
| Email sender | `no-reply@ga.esnlisboa.org` |
| DNS provider for `esnlisboa.org` | DreamHost |

## Code changes

### Back-end configuration

- `back-end/deploy/environments.ts`
  - `PROJECT`, `DOMAIN` and `PROD_CUSTOM_DOMAIN` point to the ESN Lisboa project and domain.
  - `frontEndCertificateARN` uses the ESN Lisboa ACM certificate in `us-east-1` (required by CloudFront).
  - New parameters `sesRegion` and `sesDomain` choose the SES identity used to send emails.
- `back-end/deploy.sh`: `AWS_PROFILE` and `PROJECT` use the ESN Lisboa values.

### Email sending through SES in Frankfurt

Originally, the Lambda functions sent emails through SES in the back-end region, from `no-reply@<last two parts of the API domain>`.
For this deployment that meant `no-reply@esnlisboa.org` in Ireland, where the domain identity isn't verified and the account is in the SES sandbox.
Every email failed with `MessageRejected: Email address is not verified`.

The verified, production-enabled identity is `ga.esnlisboa.org` in Frankfurt, so:

- `back-end/deploy/api-stack.ts` sets the Lambda environment variables `SES_REGION`, `SES_IDENTITY_ARN` and `SES_SOURCE_ADDRESS` from `sesRegion` and `sesDomain`.
- `back-end/deploy/main.ts` passes those parameters to the API stack.
- The handlers that send emails (`answers`, `applications`, `configurations`, `questions`, `vote`, `votingSessions`) create the SES client with `SES_REGION`.
  Templates are therefore read, edited, tested and sent in the same region.

The `esn-assembly-lisboa-2-ses` stack is still deployed in Ireland; only its SNS topic for bounce notifications is used.

### Front-end release

- `front-end/release.sh`
  - `AWS_PROFILE`, `DOMAIN_PROD` and `DOMAIN_DEV` use the ESN Lisboa values.
  - The build runs `npx ionic build --prod`, so a global Ionic CLI isn't needed.
  - The CloudFront invalidation path is `/assets/i18n/*`; CloudFront only accepts `*` at the end of a path, after a `/`.
  - The invalidation command is prefixed with `MSYS_NO_PATHCONV=1`.
    Otherwise, Git Bash on Windows rewrites `/index.html` into `C:/Program Files/Git/index.html`, and CloudFront rejects it.
- `front-end/package.json`
  - `@ionic/cli` is a dev dependency (used by `release.sh`).
  - `@ionic/core` is a direct dependency, to make sure its type definitions are installed; without them, the production build fails with `TS2339` errors on Ionic components.

## Manual infrastructure setup

These aren't managed by the code, but the app depends on them.

### DNS records in DreamHost

`esnlisboa.org` is served by DreamHost, so the records that CDK creates in the Route 53 zone `esnlisboa.org` aren't used.
They must be copied into DreamHost as CNAME records:

| Host | Points to |
|---|---|
| `ga` | Front-end CloudFront distribution |
| `media.ga` | Media CloudFront distribution |
| `api.ga` | API Gateway custom domain (HTTP API) |
| `socket.ga` | API Gateway custom domain (WebSocket API) |
| `_<id>.ga`, `_<id>.api.ga`, `_<id>.media.ga`, `_<id>.socket.ga` | ACM validation records, needed to renew the SSL certificates |
| `<id>._domainkey.ga` (three records) | SES DKIM records, needed to keep `ga.esnlisboa.org` verified in SES |

The exact values are in the Route 53 hosted zone `esnlisboa.org` (and, for DKIM, in SES in Frankfurt).
Don't add NS records for `ga` pointing to CloudFront or API Gateway hostnames: they aren't nameservers and they break resolution for every `*.ga.esnlisboa.org` name.

### SES email templates in Frankfurt

Each email uses a SES template named `<template>-<stage>` (e.g. `notify-voting-confirmation-prod`), stored in the SES region.
If a template doesn't exist, SES accepts the send request and drops the email later, without any error in the Lambda logs.

The templates are created automatically from `back-end/assets/*.hbs` when they're opened in the app's configuration page.
For `prod`, all of them now exist in Frankfurt:

- `notify-new-question-prod`
- `notify-new-answer-prod`
- `notify-application-approved-prod`
- `notify-application-rejected-prod`
- `notify-voting-instructions-prod`
- `notify-voting-confirmation-prod`

When creating a new stage (e.g. `dev`), open each template in the configuration page once, then send a test email.

## Deploying

Back-end, from `back-end` (Git Bash):

```bash
./deploy.sh prod
```

To update only the Lambda functions and their configuration, without touching the domain and certificate stacks:

```bash
npm run compile
npx cdk deploy esn-assembly-lisboa-2-prod-api --exclusively --context stage=prod --profile esnlisboa
```

Front-end, from `front-end` (Git Bash):

```bash
./release.sh prod
```

## Troubleshooting

- **Site doesn't load:** check DNS with `nslookup ga.esnlisboa.org 8.8.8.8`; it should return CloudFront addresses.
- **Emails aren't delivered:** check the CloudWatch log group of the Lambda function sending them (e.g. `/aws/lambda/esn-assembly-lisboa-2_prod_votingSessions` in Ireland), and check that the template exists in SES in Frankfurt.
