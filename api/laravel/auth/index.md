# Auth

Laravel hace que la implementación de la autenticación sea muy simple. De hecho, casi todo está configurado para usted de inmediato. El archivo de configuración de autenticación se encuentra en config / auth.php, que contiene varias opciones bien documentadas para ajustar el comportamiento de los servicios de autenticación.

En esencia, las instalaciones de autenticación de Laravel están compuestas por "protecciones" y "proveedores". Las protecciones definen cómo se autentican los usuarios para cada solicitud. Por ejemplo, Laravel se envía con un protector de sesión que mantiene el estado mediante el almacenamiento de sesiones y cookies.

Los proveedores definen cómo se recuperan los usuarios de su almacenamiento persistente. Laravel se envía con soporte para recuperar usuarios usando Eloquent y el generador de consultas de base de datos. Sin embargo, puede definir proveedores adicionales según sea necesario para su aplicación.

¡No se preocupe si todo esto suena confuso ahora! Muchas aplicaciones nunca necesitarán modificar la configuración de autenticación predeterminada.

* [CRUD Generacion de Datos](./api/laravel/auth/crud-controlador-de-generación-de-datos/index)
* [Login con JWT](./api/laravel/auth/login-con-jwt/index)