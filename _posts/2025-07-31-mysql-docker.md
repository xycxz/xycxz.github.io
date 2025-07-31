---
title: MySQL - Docker
categories: [ "Docker", "MySQL", "PHP" ]
image:
  path: apache2_docker.png
  alt: MySQL/Docker preview
layout: post
media_subpath: /assets/posts/2025-06-25-apache2-docker
tags: [ "MySQL", "PHP", "Docker", "Linux", "DevOps", "Sysadmin", "Containers" ]
---

## Objective

Today I will be explaining how to configure a MySQL database and how to also connect it to our web application. I will also introduce a code flaw on purpose to show how lack of input validation and sanitization can lead to unwanted consequences (SQL injection).

To make this even easier to visualise, I will create a separate container that will be running `phpMyAdmin`, so we can have a GUI for the database. This is the last post I am doing before starting implementing `Docker Compose` and `AWS`.
## First Steps

Like usual, we will start downloading the official image of the services we want to configure. In my case I will be using version `9.4` for [MySQL](https://hub.docker.com/_/mysql) and version `5.2-apache` for [phpMyAdmin](https://hub.docker.com/_/phpmyadmin). To make sure that our containers can communicate to each other using DNS resolution and in a separate network, I will create a custom bridge network using the following command:

```bash
docker network create <NETWORK-NAME>
```

**Note**: All this process will be simplified when we implement Docker Compose on this project. Nonetheless, understanding how networking works between containers is a crucial fundamental skill, which is why we are doing it manually first.

![[Screenshot 2025-07-29 at 15.55.13.png]]

Now we can launch both containers and give them the names that we want. Because they will share the same custom network that enables DNS resolution, it will be easy for us to locate the correct containers by their names.
## Launching and Connecting the Services

For now, we won't be diving into the configuration files but rather making sure both our services are working as expected so then we can configure them as we want. For this, we will start by running both containers as we usually do:

```bash
# Database
docker run -it --network <NETWORK-NAME> --name <CUSTOM-NAME> -e MYSQL_ROOT_PASSWORD=my-secret-pw -d <IMAGE>

# Admin PHP Panel
docker run -it --network <NETWORK-NAME> --name <CUSTOM-NAME> -p 8009:80 -e PMA_HOST=<DB-HOST> -d <IMAGE>
```

**Note**: We need to first launch the database before the GUI! I will be passing the password of the root account as an environmental variable in the command, but I would not recommend to do this. In a production environment, secrets like these would be managed by an orchestration tool like Docker Secrets or Kubernetes Secrets, not passed as plain text.

![[Screenshot 2025-07-29 at 16.24.30.png]]

We can now navigate to our `phpMyAdmin` login portal and login with the root account with the provided password:

![[Pasted image 20250729162636.png]]
## Configuring and Connecting to the Database

Before configuring our database, I will be creating an user responsible for managing the database only. For that, we can navigate to `User accounts -> Add user account`. The configuration will look as it follows:

![[Screenshot 2025-07-29 at 16.37.28.png]]

This will create a new database that the created user will have full privileges on. We also need to make sure that `Host name` is set to `Any host`, since this allows the application container to connect from the Docker network. In production, it's safer to restrict access to specific IPs or use `localhost` where possible.

Let's login with this user now and create the tables needed in the database. I will be creating a `users` table that will consist of two columns: one holding the username, and the other one its password. For that, we will select the corresponding database and create the table:

![[Pasted image 20250729164729.png]]

![[Pasted image 20250730105227.png]]

Now that we defined the `schema` of our table, we just need to click on `Save`:

![[Screenshot 2025-07-30 at 10.54.27.png]]

To start filling our table with data, we can click on the `Insert` button and pass the desired data:

![[Screenshot 2025-07-30 at 10.57.48.png]]

**Note**: I am hardcoding the password of the user with a weak hashing algorithm (MD5) for demonstration purposes. Please, when working in a production environment NEVER do this. You should always use special programming functions to encrypt the passwords!

Once we passed in the data, we can click on `Go`. I will be adding extra users to make this feel a bit more realistic:

![[Pasted image 20250730110356.png]]

**Note**: You can configure the database to not allow duplicates by adding a column with the name of `id` and the type of `INT` (check the `A I` box as well to generate unique IDs). Make sure to also give this column an `index` and select the value of `PRIMARY`. Don't forget to empty your table before doing this, otherwise it will throw an error! Adding this will give us more control over the database.

With our database ready, what we need to do now is connect it to our web application that is running on the Apache container. For that, I will write the following code:

```php
<?php 

# Initializing a new database connection
$pdo = new PDO('mysql:host=<DB_CONTAINER>;dbname=<DB_NAME>', '<USER>', '<PASSWORD>', [
	PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION
]); 

?>
```

**Note**: In the connection string `mysql:host=dbxy`, we use the container's name, `dbxy`, as the hostname. This works because both containers are on the same `custom Docker network`, which provides a built-in DNS service that resolves container names to their internal IP addresses. The user and password are the ones used to connect to this specific database, so we have to make sure they match what we previously did when creating the database and the user that manages it. Additionally, we can use the `var_dump($pdo)` function to check connectivity. If it works, we should get a message like this: `object(PDO)#1 (0) {}`.

Now we can implement the backend logic and update the `index.php` file shown in my previous post. The final code will look like the following:

```php
<?php
// Start a session to store login state
session_start();

// Include your database connection file (it is on the same directory as my index.php)
require_once 'db_connect.php';

if ($_SERVER["REQUEST_METHOD"] == "POST") {

    $username = $_POST['username'];
    $password = $_POST['password'];

// The query is built by directly inserting user input, making it vulnerable to SQL Injection!
    $sql = "SELECT * FROM users WHERE user = '$username' AND password = '" . md5($password) . "'";

    try {
        // Execute the insecure query
        $stmt = $pdo->query($sql);
        $user = $stmt->fetch(PDO::FETCH_ASSOC);

        // Check if the query returned a user
        if ($user) {
            // If a user is found, the login is successful
            $_SESSION['loggedin'] = true;
            $_SESSION['username'] = $user['user'];
            echo "<h1 style='color: green; font-family: sans-serif;'>Welcome, " . htmlspecialchars($user['user']) . "! Login successful.</h1>";
        } else {
            // If no user is found, the login fails
            echo "<h2 style='color: red; font-family: sans-serif;'>Login failed. Invalid username or password.</h2>";
        }
    } catch (PDOException $e) {
        // Show any database errors (In-Band injection)
        echo "Query failed: " . $e->getMessage();
    }

} else {
?>

    <!DOCTYPE html>
    <html>
    <head>
        <title>Login</title>
    </head>
    <body>
        <h2>Please Log In</h2>
        <p>Enter your credentials to continue please...</p>

        <form action="index.php" method="post">
            
            <label for="username">Username:</label><br>
            <input type="text" id="username" name="username"><br><br>

            <label for="password">Password:</label><br>
            <input type="password" id="password" name="password"><br><br>

            <input type="submit" value="Log In">

        </form>
    </body>
    </html>

<?php
}
?>
```
## Creating the Custom Image

With the database ready and connected to our web application, we just need to make sure to create our custom image using `Dockerfile`. 

For that, first we need to export our database using the `phpMyAdmin` interface so then we can use it when the image gets built. To achieve this, select  our database, click on `Export`, and choose `SQL` format. Then we can click on `Export`:

![[Pasted image 20250730202245.png]]

![[Screenshot 2025-07-30 at 20.25.06.png]]

We need to add the following lines to our exported file to make sure it works properly:

```sql
...
<SNIP>

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8mb4 */;

CREATE DATABASE IF NOT EXISTS `<DB-NAME>`;
USE `<DB-NAME>`;

CREATE USER 'xycxz_webapp'@'%' IDENTIFIED BY 'Password123!';
GRANT ALL PRIVILEGES ON `xycxz_web`.* TO 'xycxz_webapp'@'%';
FLUSH PRIVILEGES;

...
<SNIP>
```

Now that we have our `.sql` file, we can create the `Dockerfile`:

```bash
FROM mysql:9.4

COPY <SQL-FILE> /docker-entrypoint-initdb.d/
```

**Note**: It's worth noting that with this `Dockerfile`, we aren't building a new version of the MySQL software itself. We are using a powerful feature of the official MySQL image: any `.sql` file we `COPY` into the `/docker-entrypoint-initdb.d/` directory will be automatically executed the first time the database starts. This is the standard way to initialise a new database with tables and data.
## Conclusion

This time I won't provide any kind of automation. The reason behind this is that this is the core part of the vulnerability I am trying to showcase, and I want you to fully understand it without any automated process. Don't forget to check my other posts where I set up the `FTP` and `Web Server` containers! They are required for you to fully understand my project.
## Next Steps

Now that we've created our custom Docker images using `Dockerfile`, the next step will be putting everything together but this time using `Docker Compose`, which will ease the configuration process of multiple containers, saving networking headaches, for example.

`Docker Compose` will be the last Docker feature I will use in this project. After that, I will introduce AWS concepts and also show some cool stuff we can do with the cloud!. Stay tuned for upcoming posts!
