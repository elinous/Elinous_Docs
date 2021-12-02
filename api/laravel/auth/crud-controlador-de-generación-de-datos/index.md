Documentacion modificada el: {docsify-updated}

---

# Controlador de generación de datos

Se utilizó en:
<span class="etiqueta">Cuotas sociales</span>
<span class="etiqueta">Appunto</span>
<span class="etiqueta">Kinetic</span>
<span class="etiqueta">Tobifix</span>

---

En esta documentación se detallará a un nivel simple las funciones del controlador y sus requerimientos.

---

Para poder utilizar este generador se requerirá:

* Una base de datos conectada a Laravel que siga las recomendaciones de Laravel para los nombres de tablas y columnas.

* Las relaciones de las tablas deben estar bien definidas para que el método detecte como relacionar los modelos.

* Laravel 5.5 o superior.

* Motor de Base de Datos MySQL.

* Engine INNODB.

Nota: "Los archivos generados aparecerán en la ruta: storage/app/generated_data"

---

## Primer paso

Una vez que se tengan los requerimientos anteriores se deberá utilizar el siguiente código como un inicio para el controlador
 
Con esto se determinará la base de datos a utilizar por el controlador.

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Support\Facades\Storage;
    use Illuminate\Support\Facades\DB;

    define('DATABASE_SCHEMA', env('DB_DATABASE', 'default_aca'));
    class DataGeneratorController extends Controller
    {
    function generate_models()
    {
        $tablas = armar_tablas();
        //crear_controladores($tablas);
        return crear_modelos($tablas);
    }
    }

---

## Función "armar_tablas"

Esta función obtiene los nombres de las tablas en la base de datos, como también los atributos, pivots, relaciones, timestamps, auths, PKs, tipos de datos, nombres de columnas y relaciones inversas.

    function armar_tablas()
    {
    $tablas = [];
    $query = DB::table('information_schema.tables')
        ->select('TABLE_NAME')
        ->where('TABLE_SCHEMA', DATABASE_SCHEMA)->get();
    foreach ($query as $key => $table) {
        $tablas[$table->TABLE_NAME] = [];
        $tablas[$table->TABLE_NAME]["columnas"] = [];
        $tablas[$table->TABLE_NAME]["relaciones"] = [];
        $tablas[$table->TABLE_NAME]["relaciones_inversas"] = [];
        $tablas[$table->TABLE_NAME]["pivots"] = [];
        $tablas[$table->TABLE_NAME]["auth"] = false;
        $tablas[$table->TABLE_NAME]["timestamps"] = false;
        $tablas[$table->TABLE_NAME]['tiene_pk'] = false;
    }
    $query = DB::table('information_schema.columns')
        ->select('TABLE_NAME', 'COLUMN_NAME', 'DATA_TYPE', 'COLUMN_KEY', 'COLUMN_TYPE') //COLUMN_TYPE = int(11)
        ->where('TABLE_SCHEMA', DATABASE_SCHEMA)->get();
    foreach ($query as $key => $col) {
        $columna = [];
        $columna["COLUMN_NAME"] = $col->COLUMN_NAME;
        $columna["COLUMN_KEY"] = $col->COLUMN_KEY;
        $columna["DATA_TYPE"] = $col->DATA_TYPE;
        $columna["COLUMN_TYPE"] = $col->COLUMN_TYPE;
        $tablas[$col->TABLE_NAME]["columnas"][] = $columna;

        if ($columna["COLUMN_NAME"] == 'password') {
        $tablas[$col->TABLE_NAME]["auth"] = true;
        }
        if ($columna["COLUMN_NAME"] == 'created_at' || $columna["COLUMN_NAME"] == 'updated_at') {
        $tablas[$col->TABLE_NAME]["timestamps"] = true;
        }
        if ($columna["COLUMN_KEY"] == "PRI") {
        $tablas[$col->TABLE_NAME]['tiene_pk'] = true;
        }
    }
    /**
    * relaciones
    */
    $query = DB::table('information_schema.key_column_usage')
        ->select('CONSTRAINT_NAME', 'TABLE_NAME', 'COLUMN_NAME', 'REFERENCED_TABLE_NAME', 'REFERENCED_COLUMN_NAME')
        ->whereNotNull('REFERENCED_TABLE_NAME')
        ->where('TABLE_SCHEMA', DATABASE_SCHEMA)->get();
    foreach ($query as $key => $rel) {
        $relacion = [];
        $relacion["TABLE_NAME"] = $rel->TABLE_NAME;
        $relacion["COLUMN_NAME"] = $rel->COLUMN_NAME;
        $relacion["REFERENCED_TABLE_NAME"] = $rel->REFERENCED_TABLE_NAME;
        $relacion["REFERENCED_COLUMN_NAME"] = $rel->REFERENCED_COLUMN_NAME;
        $relacion["TIPO"] = 'belongsTo';
        $tablas[$rel->TABLE_NAME]["relaciones"][] = $relacion;

        $relacion_inversa = [];
        $relacion_inversa["REFERENCED_TABLE_NAME"] = $rel->TABLE_NAME;
        $relacion_inversa["TIPO"] = 'hasMany';

        foreach ($tablas[$rel->TABLE_NAME]["columnas"] as $columna) {
        if ($columna['COLUMN_NAME'] == $rel->COLUMN_NAME && $columna["COLUMN_KEY"] == 'UNI') {
            $relacion_inversa["TIPO"] = 'hasOne';
            break;
        }
        }
        if ($tablas[$rel->TABLE_NAME]['tiene_pk']) {
        $tablas[$rel->REFERENCED_TABLE_NAME]["relaciones_inversas"][] = $relacion_inversa;
        }
    }
    foreach ($tablas as $key => $tabla) {
        $relaciones = $tabla["relaciones"];
        //$columnas = $tabla["columnas"];
        // $tiene_pk = false;
        // foreach ($columnas as $key => $columna) {
        //     if ($columna["COLUMN_KEY"] == "PRI") {
        //         $tiene_pk = true;
        //         break;
        //     }
        // }
        $tabla['es_pivot'] = false;
        if (!$tabla['tiene_pk']) {
        if (count($relaciones) == 2) {
            $tabla['es_pivot'] = true;
            $pivot0 = array(
            "REFERENCED_TABLE_NAME" => $relaciones[0]['REFERENCED_TABLE_NAME'],
            "PIVOT" => $relaciones[0]['TABLE_NAME']
            );
            $pivot1 = array(
            "REFERENCED_TABLE_NAME" => $relaciones[1]['REFERENCED_TABLE_NAME'],
            "PIVOT" => $relaciones[1]['TABLE_NAME']
            );
            $tablas[$relaciones[0]['REFERENCED_TABLE_NAME']]["pivots"][] = $pivot1;
            $tablas[$relaciones[1]['REFERENCED_TABLE_NAME']]["pivots"][] = $pivot0;
            //return $this->belongsToMany('App\Models\'.REFERENCED_TABLE_NAME, pivot);

        }
        }
    }
    return $tablas;
    }

---

## Función "crear_controladores"

Mediante esta función se crearán los distintos CRUD que necesitará el modelo, como puede ser el mostrar los datos de cierta tabla mediante el uso de IDs o "Claves Primarias".
 
También se encuentran otros métodos CRUD como eliminar, actualizar e insertar.

    function crear_controladores($tablas)
    {
    $controladores = '';
    $routes = '';
    foreach ($tablas as $key => $tabla) {
        $table_name = $key;
        $model_name = model_name($table_name);
        $plural_model = plural($model_name);
        $controlador = "<?php
        \nnamespace App\Http\Controllers;
        \nuse Illuminate\Http\Request;
        \nuse App\\$model_name;
        \nclass {$model_name}Controller extends Controller
        {
        /**
        * GET all
        */
        public function index()
        {
            return response()->json(['message' => '$plural_model obtenidos', 'data' => $model_name::all()], 200);
        }\n
        /**
        * POST
        */
        public function store(Request \$request)
        {
            \$data = \$request->all();" . ($tabla['auth'] ? "\n        \$data['password'] = Hash::make(\$data['password']);" : "") . "
            \${$table_name} = $model_name::create(\$data);
            return response()->json(['message' => '$model_name creado', 'data' => \${$table_name}], 200);
        }\n
        /**
        * GET 1
        */
        public function show(\$id)
        {
            \${$table_name} = $model_name::findOrFail(\$id);
            return response()->json(['message' => '$model_name obtenido', 'data' => \${$table_name}], 200);
        }\n
        /**
        * PUT
        */
        public function update(Request \$request, \$id)
        {
            \${$table_name} = $model_name::findOrFail(\$id);
            \${$table_name}->update(\$request->all());
            return response()->json(['message' => '$model_name editado', 'data' => \${$table_name}], 200);
        }\n
        /**
        * DELETE
        */
        public function destroy(\$id)
        {
            \${$table_name} = $model_name::findOrFail(\$id);
            \${$table_name}->delete();
            return response()->json(['message' => '$model_name eliminado'], 200);
        }\n}";
        $filename = "{$model_name}Controller.php";
        $path = 'generated_data/app/Http/Controllers/' . $filename;
        Storage::disk('local')->put($path, $controlador);
        $controladores .= $controlador;
        $routes .= "Route::resource('$table_name', '{$model_name}Controller', [\n    'except' => ['edit', 'create']\n]);\n\n";
    }
    $path = 'generated_data/app/Http/Controllers/routes.txt';
    Storage::disk('local')->put($path, $routes);
    return $controladores;
    }

---

## Función "crear_modelos"

La siguiente función sirve para crear modelos de datos, tomando las tablas y las relaciones como también los atributos a utilizar.

    function crear_modelos($tablas)
    {
    $crear_relaciones = function ($tabla) {
        return crear_relaciones($tabla);
    };
    $fillable = function ($tabla) {
        return fillable($tabla);
    };
    $if = function ($condition, $true, $false) {
        return $condition ? $true : $false;
    };
    $modelos = '';
    foreach ($tablas as $key => $tabla) {
        $table_name = $key;
        $model_name = model_name($table_name);
        $modelo = <<<EOT
            <?php
            
            namespace App;
            
            use Illuminate\Database\Eloquent\Model;
            {$if($tabla['auth'], "use Illuminate\Foundation\Auth\User as Authenticatable;\n", "")}
            class $model_name extends {$if($tabla['auth'], "Authenticatable", "Model")}
            {
                protected \$table = '$table_name';
                
                protected \$fillable = [{$fillable($tabla)}
                ];
            {$if(!$tabla['tiene_pk'], "\n    protected \$primaryKey = null;\n\n    public \$incrementing = false;", '')}
            {$if($tabla['auth'], "\n    protected \$hidden = ['password'];", '')}
            {$if(!$tabla['timestamps'], "\n    public \$timestamps = false;\n", '')}
            {$crear_relaciones($tabla)}
            }
            
            EOT;

        $filename = "$model_name.php";
        $path = 'generated_data/app/' . $filename;
        Storage::disk('local')->put($path, $modelo);
        $modelos .= $modelo;
    }
    return $modelos;
    }

---

## Función "fillable"

Esta función tiene el trabajo de generar el array "fillable" el cual contiene los nombres de las columnas de todas las tablas dentro del modelo.

    function fillable($tabla)
    {
    $not_in = [
        "id",
        "created_at",
        "updated_at"
    ];
    $fillable = "";
    foreach ($tabla['columnas'] as $col) {
        if (!in_array($col["COLUMN_NAME"], $not_in)) {
        $fillable .= <<<EOT
                \n        '{$col["COLUMN_NAME"]}',
                EOT;
        }
    }
    return $fillable;
    }

---

## Función "crear_relaciones"

Esta función crea las relaciones que tendrán las tablas, revisando si las relaciones dentro de la base de datos es de "Uno a muchos", "Muchos a muchos" o "Uno a uno".
 
También revisa si es que la relación está vinculada a una tabla intermedia, igualmente traspasara las claves foráneas necesarias para las relaciones.

    function crear_relaciones($tabla)
    {
    $relaciones = "";
    foreach (($tabla['relaciones'] + $tabla['relaciones_inversas']) as $relacion) {
        $referenced_table = $relacion['REFERENCED_TABLE_NAME'];
        $model_name = model_name($referenced_table);
        $tipo = $relacion['TIPO'];
        $nombre_relacion = ($tipo == 'hasMany') ? pluralizar($referenced_table) : $referenced_table;
        $relaciones .= <<<EOT
                public function $nombre_relacion()
                {
                    return \$this->$tipo('App\\$model_name');
                }\n

            EOT;
    }
    foreach ($tabla['pivots'] as $relacion) {
        $referenced_table = $relacion['REFERENCED_TABLE_NAME'];
        $pivot = $relacion['PIVOT'];
        $model_name = model_name($referenced_table);
        $plural = pluralizar($referenced_table);
        $relaciones .= <<<EOT
                public function $plural()
                {
                    return \$this->belongsToMany('App\\$model_name', '$pivot');
                }

            EOT;
    }
    return $relaciones;
    }

---

## Función "model_name"

Esta función tiene un uso muy simple, transformando los nombres de las tablas en la base de datos a los estándares de Laravel.

    function model_name($table_name)
    {
    $separadas = explode('_', $table_name);
    $unirMayus = '';
    foreach ($separadas as $palabra) {
        $unirMayus .= ucfirst($palabra);
    }
    return $unirMayus;
    }

---

## Función "pluralizar"

Esta función permite transformar la primera palabra de una oración separada por guión bajo en su versión plural, esta función se utiliza en las relaciones que sean "Uno a muchos" o "Muchos a muchos".

    function pluralizar($pala_bra)
    {
    $separada = explode('_', $pala_bra);
    $separada[0] = plural($separada[0]);
    $unida = implode('_', $separada);
    return $unida;
    }

---

## Función "plural"

Esta función tiene el trabajo de determinar cuál sería el plural de las palabras (en español) esto con el fin de automatizar la creación de nombres, textos y relaciones.

    function plural($palabra)
    {
    $excepciones = [
        "crisis",
        "coxis"
    ];
    foreach ($excepciones as $key => $excepcion) {
        if ($palabra == $excepcion) {
        return $excepcion;
        }
    }
    $letra = substr($palabra, -1);
    $subletra = substr($palabra, -2, 1);

    $letra_vocal = false;
    $subletra_vocal = false;

    $distinto_de = 'lrndzjsx';
    $es_distinto = strpos($distinto_de, $letra) === false;

    $vocales = "aeiou";
    $letra_vocal = strpos($vocales, $letra) !== false;
    $subletra_vocal =  strpos($vocales, $subletra) !== false;

    if ($letra == 'y') {
        if ($subletra_vocal) {
        $plural = $palabra . 'es';
        } else {
        $palabra = substr($palabra, 0, -1) . 'i';
        $plural = $palabra . 's';
        }
        return $plural;
    }
    if ($letra_vocal || ($es_distinto && !$letra_vocal) || (!$subletra_vocal && !$letra_vocal && $letra != 's')) {
        $plural = $palabra . 's';
        return $plural;
    }
    if (!$letra_vocal) {
        if ($letra == 'z') {
        $palabra = substr($palabra, 0, -1) . 'c';
        }
        $plural = $palabra . 'es';
        return $plural;
    }
    //$es = 'lrndzj';

    return $palabra;
    }