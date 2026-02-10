Installing PostgreSQL 18 via Docker on WSL2 is straightforward once your environment is set up. While 
PostgreSQL 18 is the latest major track, ensure you pull the correct tag from Docker Hub. 

## 1. Prerequisites
Enable WSL2: Ensure you have a Linux distribution (like Ubuntu or Debian) installed. Run wsl --install in PowerShell if you haven't yet.
Install Docker Desktop: Download and install Docker Desktop for Windows.
Enable WSL Integration: In Docker Desktop settings, go to Resources > WSL Integration and toggle your Linux distro (e.g., Ubuntu) to On. 

## 2. Deployment Steps
Open your WSL terminal (Ubuntu or Debian) and run the following:
Pull the Image:
Get the specific PostgreSQL 18 image.

``` bash
sudo docker pull postgres
```

Run the Container:
Use this command to start the database. Replace mysecretpassword with your desired password.

``` bash
sudo docker run --name postgres \
  -e POSTGRES_PASSWORD=postgres \
  -v /home/postgres:/var/lib/postgresql \
  -p 5432:5432 \
  -d postgres
```

1. --name: Assigns a name to your container.
2. -e POSTGRES_PASSWORD: Sets the required superuser password.
3. -p 5432:5432: Maps the container's port to your WSL/Windows host port.
4. -d: Runs the container in the background (detached mode). 
