<?php
/*
=====================================================
   	  Модуль "Hide" v.5.4
-----------------------------------------------------
 http://cmska.org/
-----------------------------------------------------
E-mail: admin.gauss@gmail.com, admin@cmska.org
=====================================================
 Данный код защищен авторскими правами
=====================================================
 Файл: hide.class.php
-----------------------------------------------------
 Назначение: Работа скрипта
=====================================================
*/
include(ENGINE_DIR."/data/hide.php");

class dle_hide{
  private $type = false;
  private $avtor = '';
  private $text = '';
  private $dle_conf = array();
  private $hide_conf = array();
  private $member_id = array();
  private $user_group = array();
  private $is_logged = false;
  private $skin_dir = '';
  private $area = 'post';

  //////////////////////////////////////////////////////////////
  // Подхватываем все необходимые данные при запуске класса
  public final function __construct( $attr=array() ){
    global $config, $member_id, $user_group, $is_logged, $HIDEconfig;
    $this->dle_conf = (is_array($config)) ? $config : array();
    $this->hide_conf = (is_array($HIDEconfig)) ? $HIDEconfig : array();
    $this->member_id = (is_array($member_id)) ? $member_id : array();
    $this->is_logged = $is_logged;
    $this->user_group = (is_array($user_group)) ? $user_group : array();
    $this->skin_dir = ROOT_DIR.'/templates/'.$this->dle_conf['skin'].'/hide';
    if( $attr['area']=='comments' ){ $this->area = 'comments'; }
  }

  //////////////////////////////////////////////////////////////
  // Общий метод парсера, проверка основных параметров
  public final function parse($post='', $avtor=''){
    $start = false;
    if( !$post or $post=='' ){ return $post; }
    if( strpos($post, '[hide=')!==false ){ $start = true; }
    if( strpos($post, '[hidden=')!==false ){ $start = true; }
    if( !$start ){ return $post; }

    $this->avtor = $avtor;
    $this->type = 'hide';
    $post = preg_replace_callback('!\[hide=([0-9,]+)\](.+?)\[\/hide\]!i', array(&$this, 'parse_hide'), $post);
    $this->type = 'hidden';
    $post = preg_replace_callback('!\[hidden=([0-9,]+)\](.+?)\[\/hidden\]!i', array(&$this, 'parse_hide'), $post);
    return $post;
  }

  //////////////////////////////////////////////////////////////
  // Парсер тегов в шаблонах
  private final function skin_tags($param=array(0,0,0), $skin_tpl){
    for($i=0;$i<=2;$i++){ $param[$i]=intval($param[$i]); }
    $param[3]=intval($this->hide_conf['reg_bann']);

    $vip_groups = array();
    foreach($this->hide_conf['vip_group'] as $gr){
      $vip_groups[]='<b>'.$this->user_group[$gr]['group_name'].'</b>';
      $gr = null;
    } $vip_groups = implode(', ', $vip_groups);

    $ban_groups = array();
    foreach($this->hide_conf['banned_group'] as $gr){
      $ban_groups[]='<b>'.$this->user_group[$gr]['group_name'].'</b>';
      $gr = null;
    } $ban_groups = implode(', ', $ban_groups);

      $skin = $this->load_skin($skin_tpl);
      if(!$skin) die("Невозможно загрузить файл ".$skin_tpl);
      // $param[0]
      if($param[0]>0 and $this->hide_conf['field'] and $this->hide_conf['N_used']){
        $skin = str_replace('[n-param]', '', $skin);
        $skin = str_replace('[/n-param]', '', $skin);
        $skin = str_replace('{n-param}', $param[0], $skin);
      }else{ $skin = preg_replace('!\[n-param\](.+?)\[\/n-param\]!is', '', $skin); }
      // $param[1]
      if($param[1]>0){
        $skin = str_replace('[news]', '', $skin);
        $skin = str_replace('[/news]', '', $skin);
        $skin = str_replace('{news}', $param[1], $skin);
      }else{ $skin = preg_replace('!\[news\](.+?)\[\/news\]!is', '', $skin); }
      // $param[2]
      if($param[2]>0){
        $skin = str_replace('[comment]', '', $skin);
        $skin = str_replace('[/comment]', '', $skin);
        $skin = str_replace('{comment}', $param[2], $skin);
      }else{ $skin = preg_replace('!\[comment\](.+?)\[\/comment\]!is', '', $skin); }
      // $param[3]
      if($param[3]>0){
        $skin = str_replace('[date]', '', $skin);
        $skin = str_replace('[/date]', '', $skin);
        $skin = str_replace('{date}', $this->tday($param[3]), $skin);
      }else{ $skin = preg_replace('!\[date\](.+?)\[\/date\]!is', '', $skin); }

      if($this->avtor and strlen($this->avtor)>1){
        $skin = str_replace('[avtor]', '', $skin);
        $skin = str_replace('[/avtor]', '', $skin);
        $skin = str_replace('{avtor}', $this->avtor, $skin);
      }
      else{ $skin = preg_replace('!\[avtor\](.+?)\[\/avtor\]!is', '', $skin); }

      $skin = str_replace('{user_news}', $this->member_id['news_num'], $skin);
      $skin = str_replace('{user_comment}', $this->member_id['comm_num'], $skin);
      $skin = str_replace('{user_date}', $this->tday(intval((time()-$this->member_id['reg_date'])/(60*60*24))), $skin);
      $skin = str_replace('{user_n}', intval($this->member_id[$this->hide_conf['field']]), $skin);
      $skin = str_replace('{n_min}', $this->hide_conf['N_min'], $skin);
      $skin = str_replace('{text}', $this->text, $skin);
      $skin = str_replace('{vip-group}', $vip_groups, $skin);
      $skin = str_replace('{banned-group}', $ban_groups, $skin);
      $skin = str_replace('{user-group}', $this->user_group[$this->member_id['user_group']]['group_name'], $skin);

    return $skin;
  }

  //////////////////////////////////////////////////////////////
  // Непосредственно обработчик тегов хайда
  private final function parse_hide($array){
    $this->text = trim($array[2]);
    $param = explode(',', trim($array[1]));
    for($i=0;$i<=2;$i++){ $param[$i]=intval($param[$i]); }
    $param[3]=intval($this->hide_conf['reg_bann']);
    // $param[0] - N-параметр
    // $param[1] - публикации
    // $param[2] - комментарии
    // $param[3] - стаж на сайте


    // Если хайд выключен вообще
    if( !$this->hide_conf['run_mod'] ){
      return $this->text;
    }

    // Если хайд в комментах выключен
    if( $this->area == 'comments' and !$this->hide_conf['comm'] ){
      return $this->text;
    }

    // Незалогинен
    if(!$this->is_logged){
      $skin = $this->skin_tags($param, 'no_logged.tpl');
      return $skin;
    }

    // Автор новости
    if($this->member_id['name']==$this->avtor){
      $skin = $this->skin_tags($param, 'look.tpl');
      return $skin;
    }

    // banned группа
    if(!count($this->hide_conf['banned_group'])) $this->hide_conf['banned_group'] = array(5);
    if( in_array($this->member_id['user_group'], $this->hide_conf['banned_group']) ){
      $skin = $this->skin_tags($param, 'banned.tpl');
      return $skin;
    }

    // VIP группа
    if(!count($this->hide_conf['vip_group'])) $this->hide_conf['vip_group'] = array(1);
    if( in_array($this->member_id['user_group'], $this->hide_conf['vip_group']) ){
      $skin = $this->skin_tags($param, 'vip.tpl');
      return $skin;
    }

    // Проверка на низкий/высокий N-параметр
    if( $this->hide_conf['N_used'] and $this->hide_conf['field'] ){
      if($this->hide_conf['N_min']>=intval($this->member_id[$this->hide_conf['field']]) and $this->hide_conf['N_neg']){
        $skin = $this->skin_tags($param, 'negativ.tpl');
        return $skin;
      }
      if( intval($this->member_id[$this->hide_conf['field']])>$this->hide_conf['N_max'] and $this->hide_conf['N_pos'] ){
        $skin = $this->skin_tags($param, 'positiv.tpl');
        return $skin;
      }
    }

    $_p = array();

    // Проверяем N-параметр
    if($this->hide_conf['field'] and $this->hide_conf['N_used'] and $param[0]!=0){
      if(intval($this->member_id[$this->hide_conf['field']])>=$param[0]){
        $_p[0] = true;
      }else{
        $_p[0] = false;
      }
    }

    // Проверяем публикации
    if( $param[1]!=0 ){
      if($this->member_id['news_num']>=$param[1]){
        $_p[1] = true;
      }else{
        $_p[1] = false;
      }
    }

    // Проверяем комментарии
    if( $param[2]!=0 ){
      if($this->member_id['comm_num']>=$param[2]){
        $_p[2] = true;
      }else{
        $_p[2] = false;
      }
    }

    // Стаж пользователя
    if( $param[3]!=0 ){
      if($this->member_id['reg_date']<=(time()-$param[3]*60*60*24)){
        $_date = true;
      }else{
        $_date = false;
      }
    }else{ $_date = true; }

    // Если все 3 параметра равны нулю, то смотреть могут только vip
    if( !count($_p) ){
      $skin = $this->skin_tags($param, 'vip_only.tpl');
      return $skin;
    }

    // Если хайд 1-го типа и у юзера всё хорошо...
    if( in_array(true, $_p) and $this->type == 'hide' and $_date ){
      $skin = $this->skin_tags($param, 'hide_1.tpl');
      return $skin;
    }
    // Если хайд 2-го типа и у юзера всё хорошо...
    elseif( !in_array(false, $_p) and $this->type == 'hidden' and $_date ){
      $skin = $this->skin_tags($param, 'hide_2.tpl');
      return $skin;
    }
    // Если у юзера всё плохо...
    else{
      if($this->type == 'hide'){
        $skin = $this->skin_tags($param, 'block_1.tpl');
      }else{
        $skin = $this->skin_tags($param, 'block_2.tpl');
      }
      return $skin;
    }
  }

  //////////////////////////////////////////////////////////////
  // Читалка шаблонов модуля
  private final function load_skin($file){
    $id = @fopen($this->skin_dir.'/'.$file, 'rb');
    if(!$id) return false;
    $data = fread($id, filesize($this->skin_dir.'/'.$file));
    fclose($id);
    return $data;
  }

  //////////////////////////////////////////////////////////////
  // Правильный вывод "дней"
  private final function tday($num){
    $num = (int)$num;
    $_f = $num%10;
    $t_l = (int)substr($num, intval(strlen($num)-2),2 );
    if( $_f==0 or ($_f>=5 and $_f<=9) or ($t_l>=11 and $t_l<=14) ) return '<b>'.$num.'</b> дней';
    elseif($_f==1) return $num.' день';
    elseif($_f>=2 and $_f<=4) return $num.' дня';
    else{ return $num.' дней'; }
  }

  //////////////////////////////////////////////////////////////
  // Кнопка на панельке ББ-кодов
  public final function BBcode( $area ){
    $JS = '
function ins_hide( area )
{
    var enterNP   = prompt("Необходимое значение доп. параметра", "'.$this->hide_conf['fb'][0].'");
    var enterNN   = prompt("Необходимо новостей", "'.$this->hide_conf['fb'][1].'");
    var enterNC   = prompt("Необходимо комментариев", "'.$this->hide_conf['fb'][2].'");
    if (!enterNP) { enterNP ='.$this->hide_conf['fb'][0].'; }
    if (!enterNN) { enterNN ='.$this->hide_conf['fb'][1].'; }
    if (!enterNC) { enterNC ='.$this->hide_conf['fb'][2].'; }

    if( area=="hide" || area=="" ){
      doInsert("[hide="+enterNP+","+enterNN+","+enterNC+"]", "[/hide]", true);
      return false;
    }
    if( area=="hidden" ){
      doInsert("[hidden="+enterNP+","+enterNN+","+enterNC+"]", "[/hidden]", true);
      return false;
    }
}';
    $BBcode = '';
    if( !$this->hide_conf['run_mod'] or !$this->hide_conf['fastbutton'] ) return $BBcode;
    if( $area=='comments' and $this->hide_conf['comm'] ){
      $BBcode='<div class="editor_button" onclick="ins_hide(\'hide\')"><img title="HIDE" src="/engine/skins/bbcodes/images/hide1.gif" width="23" height="25" border="0"></div>
           <div class="editor_button" onclick="ins_hide(\'hidden\')"><img title="HIDDEN" src="/engine/skins/bbcodes/images/hide2.gif" width="23" height="25" border="0"></div>';
    }elseif( $area=='addnews' ){
      $BBcode='<div class="editor_button" onclick="ins_hide(\'hide\')"><img title="HIDE" src="/engine/skins/bbcodes/images/hide1.gif" width="23" height="25" border="0"></div>
           <div class="editor_button" onclick="ins_hide(\'hidden\')"><img title="HIDDEN" src="/engine/skins/bbcodes/images/hide2.gif" width="23" height="25" border="0"></div>';
    }elseif( $area=='admin' ){
      $BBcode='<div class="editor_button" onclick="ins_hide(\'hide\')"><img title="HIDE" src="/engine/skins/bbcodes/images/hide1.gif" width="23" height="25" border="0"></div>
           <div class="editor_button" onclick="ins_hide(\'hidden\')"><img title="HIDDEN" src="/engine/skins/bbcodes/images/hide2.gif" width="23" height="25" border="0"></div>';
    }elseif( $area=='js' ){
      return $JS;
    }
    return $BBcode;
  }
}

?>
