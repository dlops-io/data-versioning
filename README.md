# 🧀 Data Versioning with DVC

A hands-on tutorial for versioning the Cheese App image dataset with **DVC**, **Git**, **Docker**, and **Google Cloud Storage (GCS)**.

You’ll use Docker to run DVC in a consistent development environment, store the actual image files in GCS, and use Git tags to identify specific versions of the dataset.

## What you’ll build

The goal: **create reproducible versions of a dataset.**

You will do this in two rounds:

1. 🛠️ **Set up your own repository and cloud storage.** Create a writable copy of the class repository, create a GCP service account, and prepare a GCS bucket.

2. 🐳 **Start a DVC development container.** Use Docker to run the tools and mount your GCS image folder inside the container.

3. 📦 **Create the first dataset version.** Track the original Cheese App dataset with DVC, push it to GCS, and tag it in Git as `dataset_v20`.

4. 🔄 **Update the dataset and create a new version.** Add images, update DVC tracking, and tag the new state as `dataset_v21`.

By the end, you will be able to retrieve a specific version of the dataset instead of relying on whichever files happen to be in the bucket today.

## Prerequisites

Complete the following before starting:

- Install and start the latest version of Docker Desktop.
- Have a GCP account and project.
- Have access to a GCS bucket.
- Have Git and a GitHub account.
- Configure GitHub SSH access if you use the SSH commands below.

> You need a GCP service-account key for this tutorial. Keep it private and never commit it to GitHub.

---

## Contents

Four walkthroughs:

- [Prepare Docker and create your repository](#-prepare-docker-and-create-your-repository)
- [Set up GCP storage and credentials](#-set-up-gcp-storage-and-credentials)
- [Start the DVC container](#-start-the-dvc-container)
- [Create and view dataset versions](#-create-and-view-dataset-versions)

---

# 🛠️ Prepare Docker and Create Your Repository

**Step 1 of 4 — prepare a clean local development environment.** First, make sure Docker is ready. Then create your own GitHub repository, since the class repository only gives you read access.

## Make sure we do not have any running containers and clear up unused images

- Run:

  ```bash
  docker container ls
  ```

- Stop any container that is running.

- Run:

  ```bash
  docker system prune
  ```

- Run:

  ```bash
  docker image ls
  ```

> 💡 **Tip:** `docker system prune` removes unused Docker resources. Read the confirmation prompt before entering `y`.

## Clone the class repository

Clone the `data-versioning-ac215` branch:

```bash
git clone -b data-versioning-ac215 git@github.com:dlops-io/data-versioning.git
```

> 💡 **Tip:** This command uses SSH. You need to add your public SSH key to GitHub first. Alternatively, you can clone with HTTPS.

Move into the project folder:

```bash
cd data-versioning
```

Remove the existing Git metadata:

```bash
rm -rf .git
```

Initialize a new Git repository:

```bash
git init
```

> ⚠️ **Why remove `.git`?** The cloned repository is connected to the original class repository. Removing `.git` disconnects it so you can connect the code to a repository that you own.

## Create your own private GitHub repository

Create a new **private** repository on GitHub called `data-versioning`.

Do not add a README, license, or `.gitignore` when creating it because these files already exist locally.

Your repository URL should look like this:

```text
git@github.com:YOUR_GITHUB_USERNAME/data-versioning.git
```

Connect your local project to the new repository and push it:

```bash
git branch -M main
git remote add origin git@github.com:YOUR_GITHUB_USERNAME/data-versioning.git
git add .
git commit -m "Initial commit"
git push -u origin main
```

---

# ☁️ Set Up GCP Storage and Credentials

**Step 2 of 4 — give the container secure access to cloud storage.** You will create a service account and prepare two folders in your GCS bucket: one for dataset images and one for DVC’s versioned files.

## Add the `secrets` folder

Create a folder named `secrets` next to your `data-versioning` folder.

Your folder structure should look like this:

```text
parent-folder/
├── data-versioning/
└── secrets/
```

> 🔒 **Important:** Keeping `secrets` outside the repository helps prevent you from accidentally uploading credentials to GitHub.

## Set up a GCP service account

3. Create a new service account in the GCP console named:

   ```text
   data-service-account
   ```

4. For **Service account permissions**, choose:

   ```text
   Cloud Storage → Storage Admin
   ```

5. Click **Continue**, then **Done**.
6. In the service-account list, click the three dots (`⋮`) in the **Actions** column and select **Manage keys**.
7. Select **Add Key → Create new key → JSON**.
8. Download the JSON key file.
9. Copy it into your `secrets` folder and rename it:

   ```text
   data-service-account.json
   ```

> ⚠️ **Never commit this JSON file to GitHub.** It is a private credential that grants access to your GCP resources.

## Create folders in your GCS bucket

Go to the [GCS browser](https://console.cloud.google.com/storage/browser).

Inside your bucket, create these folders:

```text
YOUR_BUCKET_NAME/
├── dvc_store/
└── images/
```

- `images/` stores the Cheese App image dataset.
- `dvc_store/` stores the DVC-managed copies of each dataset version.

---

# 🐳 Start the DVC Container

**Step 3 of 4 — run DVC in a consistent Docker environment.** The container mounts the images in your GCS bucket into the project, so DVC can track them.

## Configure the container parameters

Inside your local `data-versioning` folder, open `docker-shell.sh`.

Replace the following values with your own setup:

```bash
export GCS_BUCKET_NAME="YOUR_BUCKET_NAME"
export GCP_PROJECT="YOUR_GCP_PROJECT_ID"
export GCP_ZONE="YOUR_GCP_ZONE"
```

For example:

```bash
export GCS_BUCKET_NAME="cheese-app-data-versioning"
export GCP_PROJECT="ac215-project"
export GCP_ZONE="us-central1-a"
```

## 🧩 About `docker-entrypoint.sh`

The project includes a file named `docker-entrypoint.sh`.

An entrypoint script automatically runs when the container starts. It performs setup tasks that must happen every time the container runs.

For this container, it:

- Mounts your GCS bucket inside the container.
- Connects the bucket’s `images` folder to:

  ```text
  /app/cheese_dataset
  ```

This makes the images stored in GCS available inside the container as the local `cheese_dataset` folder.

## Run `docker-shell.sh`

Make sure you are inside the `data-versioning` folder:

```bash
cd data-versioning
```

Start the DVC development container:

```bash
sh docker-shell.sh
```

> 💡 **Note:** Run the DVC commands in the next section inside this container unless the tutorial says otherwise.

---

# 📦 Create and View Dataset Versions

**Step 4 of 4 — use DVC and Git together to create reproducible dataset versions.**

DVC and Git have different jobs:

| Tool | What it tracks |
|---|---|
| Git | Code and small DVC metadata files |
| DVC | Large dataset files |
| GCS | The remote location where DVC stores dataset versions |

## Create the first dataset version

### Initialize DVC

Inside the container, run:

```bash
dvc init
```

This creates the DVC configuration files in your project.

### Add the GCS remote registry

Connect DVC to the `dvc_store` folder in your bucket:

```bash
dvc remote add -d cheese_dataset gs://YOUR_BUCKET_NAME/dvc_store
```

For example:

```bash
dvc remote add -d cheese_dataset gs://cheese-app-data-versioning/dvc_store
```

### Add the dataset to DVC

Tell DVC to track the dataset:

```bash
dvc add cheese_dataset
```

DVC creates a small `.dvc` metadata file describing the exact state of the dataset.

### Push the dataset to GCS

Upload the DVC-tracked dataset files:

```bash
dvc push
```

You can now open your GCS bucket and inspect the `dvc_store` folder.

> 💡 **Tip:** The actual image files are stored in GCS. Git does not need to store these large files directly.

### Update Git to track the DVC metadata

Run these commands **outside the container**, from your local `data-versioning` folder:

```bash
git status
git add .
git commit -m "Track initial cheese dataset with DVC"
git tag -a dataset_v20 -m "First version of cheese dataset"
git push --atomic origin main dataset_v20
```
> ‼️ **Disclaimer** The name-tag of the dataset might not be 'dataset_v20' for you as this tag is taken. Try with different numbers and see which might work for you for e.g. dataset_v21, dataset_v22, dataset_v23, etc. 
> 💡 **Why tag the commit?** The Git tag `dataset_v20` gives this exact dataset state a memorable name. Later, you can return to that version instead of guessing which data was used.

## View `dataset_v20` in Colab

- Open the [Colab Notebook](https://colab.research.google.com/drive/1RRQ1SlHq5lKK76R8LoQdi5LjCnND3jTq?usp=sharing).
- Follow the instructions in the notebook.
- View the dataset associated with `dataset_v20`.

---

# 🔄 Create an Updated Dataset Version: `dataset_v21`

Now, simulate a real-world dataset update by adding new images.

## Upload images

Upload a few additional images to the `images` folder in your GCS bucket:

```text
YOUR_BUCKET_NAME/images/
```

Because this folder is mounted into the container as `/app/cheese_dataset`, the new files will appear in the dataset directory.

## Add the changed dataset to DVC

Inside the container, run:

```bash
dvc add cheese_dataset
```

Push the updated dataset to GCS:

```bash
dvc push
```

## Update Git and create `dataset_v21`

Run these commands **outside the container**:

```bash
git status
git add .
git commit -m "Add images to cheese dataset"
git tag -a dataset_v21 -m "Updated cheese dataset with additional images"
git push --atomic origin main dataset_v21
```
> ‼️ **Disclaimer** Again, he name-tag of the dataset might not be 'dataset_v21' for you as this tag is taken. Try with different numbers and see which might work for you for e.g. dataset_v22, dataset_v23, dataset_v24, etc. 
> ⚠️ **Note:** You only need to run `git remote add origin ...` once, when you first create your repository. Do not add it again here.

You now have two dataset versions:

| Git Tag | Dataset State |
|---|---|
| `dataset_v20` | Original Cheese App image dataset |
| `dataset_v21` | Dataset with additional uploaded images |

## View `dataset_v21` in Colab

- Open the [Colab Notebook](https://colab.research.google.com/drive/1RRQ1SlHq5lKK76R8LoQdi5LjCnND3jTq?usp=sharing).
- Follow the instructions in the notebook.
- View the dataset associated with `dataset_v21`.

---

# 🎉 What You Learned

You used Git and DVC together to version a dataset:

- DVC tracked the dataset contents.
- GCS stored the large image files.
- Git stored the lightweight DVC metadata.
- Git tags identified meaningful dataset versions.
- Colab let you retrieve and inspect a specific version.

This workflow makes ML experiments reproducible because you can always identify exactly which dataset version was used.

---

# 🧹 Docker Cleanup

When you are done, clean up Docker resources:

- Run:

  ```bash
  docker container ls
  ```

- Stop any container that is running.

- Run:

  ```bash
  docker system prune
  ```

- Run:

  ```bash
  docker image ls
  ```# 🧀 Data Versioning with DVC

A hands-on tutorial for versioning the Cheese App image dataset with **DVC**, **Git**, **Docker**, and **Google Cloud Storage (GCS)**.

You’ll use Docker to run DVC in a consistent development environment, store the actual image files in GCS, and use Git tags to identify specific versions of the dataset.

## What you’ll build

The goal: **create reproducible versions of a dataset.**

You will do this in two rounds:

1. 🛠️ **Set up your own repository and cloud storage.** Create a writable copy of the class repository, create a GCP service account, and prepare a GCS bucket.

2. 🐳 **Start a DVC development container.** Use Docker to run the tools and mount your GCS image folder inside the container.

3. 📦 **Create the first dataset version.** Track the original Cheese App dataset with DVC, push it to GCS, and tag it in Git as `dataset_v20`.

4. 🔄 **Update the dataset and create a new version.** Add images, update DVC tracking, and tag the new state as `dataset_v21`.

By the end, you will be able to retrieve a specific version of the dataset instead of relying on whichever files happen to be in the bucket today.

## Prerequisites

Complete the following before starting:

- Install and start the latest version of Docker Desktop.
- Have a GCP account and project.
- Have access to a GCS bucket.
- Have Git and a GitHub account.
- Configure GitHub SSH access if you use the SSH commands below.

> You need a GCP service-account key for this tutorial. Keep it private and never commit it to GitHub.

---

## Contents

Four walkthroughs:

- [Prepare Docker and create your repository](#-prepare-docker-and-create-your-repository)
- [Set up GCP storage and credentials](#-set-up-gcp-storage-and-credentials)
- [Start the DVC container](#-start-the-dvc-container)
- [Create and view dataset versions](#-create-and-view-dataset-versions)

---

# 🛠️ Prepare Docker and Create Your Repository

**Step 1 of 4 — prepare a clean local development environment.** First, make sure Docker is ready. Then create your own GitHub repository, since the class repository only gives you read access.

## Make sure we do not have any running containers and clear up unused images

- Run:

  ```bash
  docker container ls
  ```

- Stop any container that is running.

- Run:

  ```bash
  docker system prune
  ```

- Run:

  ```bash
  docker image ls
  ```

> 💡 **Tip:** `docker system prune` removes unused Docker resources. Read the confirmation prompt before entering `y`.

## Clone the class repository

Clone the `data-versioning-ac215` branch:

```bash
git clone -b data-versioning-ac215 git@github.com:dlops-io/data-versioning.git
```

> 💡 **Tip:** This command uses SSH. You need to add your public SSH key to GitHub first. Alternatively, you can clone with HTTPS.

Move into the project folder:

```bash
cd data-versioning
```

Remove the existing Git metadata:

```bash
rm -rf .git
```

Initialize a new Git repository:

```bash
git init
```

> ⚠️ **Why remove `.git`?** The cloned repository is connected to the original class repository. Removing `.git` disconnects it so you can connect the code to a repository that you own.

## Create your own private GitHub repository

Create a new **private** repository on GitHub called `data-versioning`.

Do not add a README, license, or `.gitignore` when creating it because these files already exist locally.

Your repository URL should look like this:

```text
git@github.com:YOUR_GITHUB_USERNAME/data-versioning.git
```

Connect your local project to the new repository and push it:

```bash
git branch -M main
git remote add origin git@github.com:YOUR_GITHUB_USERNAME/data-versioning.git
git add .
git commit -m "Initial commit"
git push -u origin main
```

---

# ☁️ Set Up GCP Storage and Credentials

**Step 2 of 4 — give the container secure access to cloud storage.** You will create a service account and prepare two folders in your GCS bucket: one for dataset images and one for DVC’s versioned files.

## Add the `secrets` folder

Create a folder named `secrets` next to your `data-versioning` folder.

Your folder structure should look like this:

```text
parent-folder/
├── data-versioning/
└── secrets/
```

> 🔒 **Important:** Keeping `secrets` outside the repository helps prevent you from accidentally uploading credentials to GitHub.

## Set up a GCP service account

1. Open the [GCP Console](https://console.cloud.google.com/home/dashboard).
2. Search for **Service Accounts**, or navigate to **IAM & Admin → Service Accounts**.
3. Create a new service account named:

   ```text
   data-service-account
   ```

4. For **Service account permissions**, choose:

   ```text
   Cloud Storage → Storage Admin
   ```

5. Click **Continue**, then **Done**.
6. In the service-account list, click the three dots (`⋮`) in the **Actions** column and select **Manage keys**.
7. Select **Add Key → Create new key → JSON**.
8. Download the JSON key file.
9. Copy it into your `secrets` folder and rename it:

   ```text
   data-service-account.json
   ```

> ⚠️ **Never commit this JSON file to GitHub.** It is a private credential that grants access to your GCP resources.

## Create folders in your GCS bucket

Go to the [GCS browser](https://console.cloud.google.com/storage/browser).

Inside your bucket, create these folders:

```text
YOUR_BUCKET_NAME/
├── dvc_store/
└── images/
```

- `images/` stores the Cheese App image dataset.
- `dvc_store/` stores the DVC-managed copies of each dataset version.

---

# 🐳 Start the DVC Container

**Step 3 of 4 — run DVC in a consistent Docker environment.** The container mounts the images in your GCS bucket into the project, so DVC can track them.

## Configure the container parameters

Inside your local `data-versioning` folder, open `docker-shell.sh`.

Replace the following values with your own setup:

```bash
export GCS_BUCKET_NAME="YOUR_BUCKET_NAME"
export GCP_PROJECT="YOUR_GCP_PROJECT_ID"
export GCP_ZONE="YOUR_GCP_ZONE"
```

For example:

```bash
export GCS_BUCKET_NAME="cheese-app-data-versioning"
export GCP_PROJECT="ac215-project"
export GCP_ZONE="us-central1-a"
```

## 🧩 About `docker-entrypoint.sh`

The project includes a file named `docker-entrypoint.sh`.

An entrypoint script automatically runs when the container starts. It performs setup tasks that must happen every time the container runs.

For this container, it:

- Mounts your GCS bucket inside the container.
- Connects the bucket’s `images` folder to:

  ```text
  /app/cheese_dataset
  ```

This makes the images stored in GCS available inside the container as the local `cheese_dataset` folder.

## Run `docker-shell.sh`

Make sure you are inside the `data-versioning` folder:

```bash
cd data-versioning
```

Start the DVC development container:

```bash
sh docker-shell.sh
```

> 💡 **Note:** Run the DVC commands in the next section inside this container unless the tutorial says otherwise.

---

# 📦 Create and View Dataset Versions

**Step 4 of 4 — use DVC and Git together to create reproducible dataset versions.**

DVC and Git have different jobs:

| Tool | What it tracks |
|---|---|
| Git | Code and small DVC metadata files |
| DVC | Large dataset files |
| GCS | The remote location where DVC stores dataset versions |

## Create the first dataset version: `dataset_v20`

### Initialize DVC

Inside the container, run:

```bash
dvc init
```

This creates the DVC configuration files in your project.

### Add the GCS remote registry

Connect DVC to the `dvc_store` folder in your bucket:

```bash
dvc remote add -d cheese_dataset gs://YOUR_BUCKET_NAME/dvc_store
```

For example:

```bash
dvc remote add -d cheese_dataset gs://cheese-app-data-versioning/dvc_store
```

### Add the dataset to DVC

Tell DVC to track the dataset:

```bash
dvc add cheese_dataset
```

DVC creates a small `.dvc` metadata file describing the exact state of the dataset.

### Push the dataset to GCS

Upload the DVC-tracked dataset files:

```bash
dvc push
```

You can now open your GCS bucket and inspect the `dvc_store` folder.

> 💡 **Tip:** The actual image files are stored in GCS. Git does not need to store these large files directly.

### Update Git to track the DVC metadata

Run these commands **outside the container**, from your local `data-versioning` folder:

```bash
git status
git add .
git commit -m "Track initial cheese dataset with DVC"
git tag -a dataset_v20 -m "First version of cheese dataset"
git push --atomic origin main dataset_v20
```

> 💡 **Why tag the commit?** The Git tag `dataset_v20` gives this exact dataset state a memorable name. Later, you can return to that version instead of guessing which data was used.

## View `dataset_v20` in Colab

- Open the [Colab Notebook](https://colab.research.google.com/drive/1RRQ1SlHq5lKK76R8LoQdi5LjCnND3jTq?usp=sharing).
- Follow the instructions in the notebook.
- View the dataset associated with `dataset_v20`.

---

# 🔄 Create an Updated Dataset Version: `dataset_v21`

Now, simulate a real-world dataset update by adding new images.

## Upload images

Upload a few additional images to the `images` folder in your GCS bucket:

```text
YOUR_BUCKET_NAME/images/
```

Because this folder is mounted into the container as `/app/cheese_dataset`, the new files will appear in the dataset directory.

## Add the changed dataset to DVC

Inside the container, run:

```bash
dvc add cheese_dataset
```

Push the updated dataset to GCS:

```bash
dvc push
```

## Update Git and create `dataset_v21`

Run these commands **outside the container**:

```bash
git status
git add .
git commit -m "Add images to cheese dataset"
git tag -a dataset_v21 -m "Updated cheese dataset with additional images"
git push --atomic origin main dataset_v21
```

> ⚠️ **Note:** You only need to run `git remote add origin ...` once, when you first create your repository. Do not add it again here.

You now have two dataset versions:

| Git Tag | Dataset State |
|---|---|
| `dataset_v20` | Original Cheese App image dataset |
| `dataset_v21` | Dataset with additional uploaded images |

## View `dataset_v21` in Colab

- Open the [Colab Notebook](https://colab.research.google.com/drive/1RRQ1SlHq5lKK76R8LoQdi5LjCnND3jTq?usp=sharing).
- Follow the instructions in the notebook.
- View the dataset associated with `dataset_v21`.

---

# 🎉 What You Learned

You used Git and DVC together to version a dataset:

- DVC tracked the dataset contents.
- GCS stored the large image files.
- Git stored the lightweight DVC metadata.
- Git tags identified meaningful dataset versions.
- Colab let you retrieve and inspect a specific version.

This workflow makes ML experiments reproducible because you can always identify exactly which dataset version was used.

---

# 🧹 Docker Cleanup

When you are done, clean up Docker resources:

- Run:

  ```bash
  docker container ls
  ```

- Stop any container that is running.

- Run:

  ```bash
  docker system prune
  ```

- Run:

  ```bash
  docker image ls
  ```
