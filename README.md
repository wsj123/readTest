<?php

namespace app\controllers;

use app\components\BaseController;
use app\models\GameAlias;
use app\models\Subject;
use yii\filters\AccessControl;
use yii\filters\VerbFilter;
use yii\helpers\Json;
use yii\web\HttpException;
use app\models\Activity;
use Yii;

class GameController extends BaseController
{
    /**
     * @inheritdoc
     */
    public function behaviors()
    {
        return [
            'access' => [
                'class' => AccessControl::className(),
                'only' => ['logout'],
                'rules' => [
                    [
                        'actions' => ['logout'],
                        'allow' => true,
                        'roles' => ['@'],
                    ],
                ],
            ],
            'verbs' => [
                'class' => VerbFilter::className(),
                'actions' => [
                    'logout' => ['post'],
                ],
            ],
        ];
    }

    /**
     * @inheritdoc
     */
    public function actions()
    {
        return [
            'error' => [
                'class' => 'yii\web\ErrorAction',
            ]
        ];
    }

    /**
     * Displays homepage.
     *
     * @return string
     */
    public function actionIndex($tag,$bei,$wen,$theme,$id,$alias_id=null)
    {
        // 设置key为id的md5值
        $alias_string = $alias_id == null ? '' : $alias_id;
        $redisKey =  md5($id.$bei.$wen.$theme.$tag.$alias_string);
        $redis    = Yii::$app->redis;
        $data     = $redis->get($redisKey);
        if (!$data) {
            $data = Activity::find()
                ->select(['download_url','referer','is_down','alias_info'])
                ->where(['substring(referer,9)'=>$id])
                //->andWhere(['is_package'=>2])
                ->asArray()->one();
            if($data) {
                $data['beian'] = "";
                $data['wen']   = "";
                $beiRet = Subject::findOne($bei);
                $wenRet = Subject::findOne($wen);
                if($beiRet){
                    $data['beian'] = $beiRet->name;
                }
                if($wenRet){
                    $data['wen'] = $wenRet->name;
                }
                // 获取别名信息
                $aliasInfo = GameAlias::findOne(['id'=>$alias_id]);
                $data['alias'] = $data['icon'] = '';
                if ($aliasInfo) {
                    $data['alias'] = $aliasInfo['alias'];
                    $data['icon'] = $aliasInfo['icon'];
                }

                $data['id'] = $id;
                $data = Json::encode($data);
                $redis->set($redisKey, $data);
                $redis->expire($redisKey, 60);
            }
        }

        $activeData = array('download_url'=>'','wen'=>'','beian'=>'','referer'=>'','is_down'=>0,'alias'=>'','icon'=>'','id'=>0);
        $tmpData = Json::decode($data);
        if ($tmpData){
            $activeData = $tmpData;
        }

        // 控制某些特定的落地页进入并跳转
        $inRedirect  = array(
            // '1000016_2006',
            // '1000016_2007',
            // '1000016_2008',
            // '1000016_2009',
            // '1000016_2010',
            // '1000016_2011',
            // '1000016_2012',
            // '1000016_2013',
            // '1000016_2014',
            // '1000016_2015',
            // '1000016_2016',
            // '1000016_2017'
        );
        $isRedirect = false;
        if ((in_array($activeData['referer'],$inRedirect)) OR $activeData['is_down']==1){
            $isRedirect = true;
            return $this->redirect($activeData['download_url']);
        }
        return $this->render('index',['activeData'=>$activeData,'isRedirect'=>$isRedirect]);
    }
}
