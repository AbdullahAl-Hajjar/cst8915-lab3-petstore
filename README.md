# CST8915 Lab 3: Algonquin Pet Store on Azure PaaS

**Student Name**: Abdullah Al-Hajjar
**Student ID**: 040656012
**Course**: CST8915 Full-stack Cloud-native Development
**Semester**: Fall 2026

---

## Demo Video

🎥 [Watch Demo Video](https://youtu.be/WjFZguuvdnM)

---

## Service Repositories

| Service | Repository | Deployed on Azure |
| --- | --- | --- |
| order-service (Node.js) | https://github.com/AbdullahAl-Hajjar/order-service | https://order-service-app-aa.azurewebsites.net (Azure Web App) |
| product-service (Python, rewritten from Rust) | https://github.com/AbdullahAl-Hajjar/product-service | https://product-service-app-aa.azurewebsites.net/products (Azure Web App) |
| store-front (Vue.js) | https://github.com/AbdullahAl-Hajjar/store-front | https://store-front-app-aa.azurewebsites.net (Azure Web App, built by GitHub Actions) |

RabbitMQ runs on its own Ubuntu 24.04 VM (`rabbitmq-vm`).

---

## How It's Deployed

- **RabbitMQ VM**: Ubuntu 24.04, Standard_B2als_v2, North Central US. RabbitMQ 4.3 with the management UI. The NSG only lets port 5672 (AMQP) in from the order-service Web App's outbound IPs, and SSH/15672 only from my own IP. The order-service logs in with its own `orderapp` account instead of `guest`.
- **order-service**: Linux Web App on Node 24 LTS, deployed from its GitHub repo. `package.json` only has the `test` script, as the lab asks, and App Service starts it with `node index.js`. The only app setting it needs is `RABBITMQ_CONNECTION_STRING`. App Service supplies `PORT` itself.
- **product-service**: rewritten in Python (Flask) because App Service can't run Rust. Same `GET /products` endpoint, same three products, same open-CORS GET policy. Linux Web App on Python 3.12, deployed from its GitHub repo. App Service installs `requirements.txt` and runs it with gunicorn.
- **store-front**: built by a GitHub Actions workflow with the two `VUE_APP_` URLs declared in an `env:` block on `build_and_deploy_job`, then deployed to a Linux Web App that serves the static `dist/` files.
- All three Web Apps share one Free (F1) Linux App Service plan in North Central US.

### Changes from the lab instructions

- **Azure Static Web Apps wasn't possible.** My Azure for Students subscription only allows five regions, and Static Web Apps is only offered in Central US, East US 2, West US 2, West Europe and East Asia. Every create attempt failed with `RequestDisallowedByAzure`. I hosted the store-front on App Service instead, deployed by GitHub Actions with the same `env:` block, so the build-time configuration works exactly the way the lab describes.
- **App names have an `-aa` suffix** (`order-service-app-aa` and so on) because `order-service-app` was already taken. Web App names are global.
- **Region is North Central US**, the region my subscription can deploy to.

---

## Reflection Questions

### 1. What challenges did you encounter when configuring environment variables in the GitHub Actions workflow?

The big mindset change was that the store-front's variables aren't read when the app runs. They get baked into the JavaScript during `npm run build`, so changing a URL means a rebuild and redeploy, not just a restart. A small one: the lab's example puts a trailing slash on the URLs, but the code adds `/orders` and `/products` itself, which would give `//orders`, so I left the slash off. The biggest one was that Static Web Apps wasn't available on my subscription, so I couldn't use the auto-generated workflow. I wrote my own workflow for App Service, and it needed a deployment credential stored as a GitHub secret (`AZURE_WEBAPP_PUBLISH_PROFILE`). That showed me the difference between public settings that can sit in the `env:` block and secrets that never belong in the repo.

### 2. How does deploying microservices on Azure Web App Service differ from running them locally?

Locally I start each service myself with `node index.js` or `python app.py`, read config from a `.env` file, and everything talks over `localhost`. On App Service I just pick a runtime (Node 24, Python 3.12), point it at the GitHub repo, and Azure builds it, starts it, gives it HTTPS and a public URL, and restarts it if it crashes. There's no VM to patch. Config moves into app settings, and App Service decides some of it itself, like `PORT` and running gunicorn for Python. You also have to play by the platform's rules: Rust isn't supported, so the product-service had to be rewritten, and the free tier sleeps when idle, so the first request after a while is slow. Networking also changes. The services are no longer on the same machine as RabbitMQ, so I had to open port 5672 on the VM to only the order-service's outbound IPs.

### 3. Why is it important to use environment variables for configurations in a cloud environment?

It keeps one codebase deployable anywhere. The same code runs on my laptop and on Azure, and only the variables change. It also keeps secrets out of the code, which matters a lot here because all of these repos are public. The RabbitMQ password only lives in the Web App's settings. Changing where a service points, like moving RabbitMQ to a different server, becomes a settings change and a restart instead of a code change and a new commit. And each microservice gets configured on its own, which is what lets them be deployed and scaled independently.
