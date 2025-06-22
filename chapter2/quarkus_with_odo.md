# Live coding with Quarkus and odo

## 🎯 Goal
Demonstrate how to initialize, develop, and expose a Quarkus application using `odo` in OpenShift.

---

## 🛠️ Step 1: Generate a Quarkus example application

Visit [code.quarkus.io](https://code.quarkus.io/) and generate a project with the following options:
- **Quarkus version**: 3.15 LTS
- **Extensions**: RESTEasy (REST application)

Then, move and unzip the downloaded project:

```bash
# Create a directory for your Quarkus project
mkdir quarkus

# Move the downloaded zip into it
mv Downloads/code-with-quarkus.zip quarkus/

# Change into the project directory
cd quarkus/

# Unzip the Quarkus project
unzip code-with-quarkus.zip

# Enter the unzipped project directory
cd code-with-quarkus/
```

---

## 🚀 Step 2: Create an OpenShift project

```bash
oc new-project quarkus
```

Creates a new OpenShift namespace called `quarkus` where your app will be deployed.

---

## ⚙️ Step 3: Initialize the odo component

```bash
odo init
```

This command scans the current directory and detects it's a **Java Quarkus** project.
- It downloads a compatible Devfile (`java-quarkus:1.5.0`)
- Opens port 8080 (application) and 5858 (debug)
- Asks you to confirm the component name (`code-with-quarkus`)

📝 Optional: You can script this with:

```bash
odo init --name code-with-quarkus --devfile java-quarkus --devfile-registry DefaultDevfileRegistry --devfile-version 1.5.0
```

---

## 🧪 Step 4: Start development mode

```bash
odo dev
```

This starts **Dev Mode** on the cluster using the Devfile.
- Syncs your local code to the container
- Runs the app live on the cluster
- Opens local web console: [http://localhost:20000/](http://localhost:20000/)
- Swagger API docs: [http://localhost:20000/swagger-ui/](http://localhost:20000/swagger-ui/)

If the app compiles, it will be available at the internal route.

---

## 🌐 Step 5: Expose the service

```bash
oc expose service code-with-quarkus-app
```

This creates a route to access your application from outside the cluster.

You can test it with:

```bash
curl code-with-quarkus-app-quarkus.apps.ocp4.example.com/hello
```

---

## ✏️ Step 6: Live updates

Modify the application using **Codium** (or any IDE) and see changes applied in real-time thanks to `odo dev` live sync.

---

Happy Coding with Quarkus + OpenShift + odo 🚀
