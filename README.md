# pgsql-stream-on-bytea
```php
$host = "localhost";
$port = 45432;
$user = "test";
$pass = "test";
$db = "test";


//drop table images;
//CREATE TABLE images (imgname text, img bytea); 

$conn = new PDO('pgsql' . ":" . "host=$host port=$port dbname=$db", $user, $pass);
$conn->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);



insert();
select();

function insert()
{
    global $conn;

    $filepath = '1468664232-john-doe.zip';
    $img = fopen($filepath, 'rb');
    try {
        $conn->beginTransaction();
        $data = $conn->pgsqlLOBCreate();
        $strm = $conn->pgsqlLOBOpen($data, 'w');
        stream_copy_to_stream($img, $strm);


        $sql = "INSERT INTO images(imgname, img) values (:name, lo_get(:data::oid)::bytea )";

        $stmt = $conn->prepare($sql);
        $stmt->bindValue(1, $filepath, PDO::PARAM_STR);
        $stmt->bindValue(2, $strm, PDO::PARAM_LOB);
        $stmt->execute();
        $conn->pgsqlLOBUnlink($data);
        $conn->commit();

    } catch (PDOException $e) {
        die('ERROR: ' . $e->getMessage() . "\n");
    }
    fclose($img);
}

function select()
{
    global $conn;
    try {
        $conn->beginTransaction();
        $name = '';

        $sql = "SELECT imgname,  lo_from_bytea(0,img) img from images where imgname='1468664232-john-doe.zip' ";

        $stmt = $conn->prepare($sql);
        $stmt->execute();
        $stmt->bindColumn(1, $name, PDO::PARAM_STR);
        $stmt->bindColumn(2, $lob, PDO::PARAM_LOB);
        $stmt->fetch(PDO::FETCH_BOUND);
        $stream = $conn->pgsqlLOBOpen($lob, 'r');
        $tmp = tmpfile();
        stream_copy_to_stream($stream, $tmp);
        $conn->pgsqlLOBUnlink($lob);
        rewind($tmp);
        $meta = stream_get_meta_data($tmp);
        var_dump(filesize($meta['uri']));
    } catch (PDOException $e) {
        die('ERROR: ' . $e->getMessage() . "\n");
    }
    fclose($tmp);
}
```
