## GTM을 통해 내부사이트 검색어 추출

### 1. Enter 키 입력을 통한 내부검색 키워드 추출
#### 1.1 GTM Tag 생성 
* Custom HTML Type Tags 생성
```javascript
<script>
  (function() {
    var el = document.querySelector('검색입력 Object');
    if(el){
      el.addEventListener('keyup', function(e) {
        if (e.keyCode === 13) {
          window.dataLayer.push({
            event: 'search',
            search_keyword_typed: jQuery(e.target).val()
          });
        }
      }, true);
    }
    var el2 = document.querySelector('#searchForm2_keyword');
    if(el2){
      el2.addEventListener('keyup', function(e) {
        if (e.keyCode === 13) {
          window.dataLayer.push({
            event: 'search',
            search_keyword_typed: jQuery(e.target).val()
          });
        }
      }, true);
    }
 </script>   
```
#### 1.2 Trigger 맵핑
* Dom Ready 트리거 생성 후 1.1 Tag와 맵핑.

## NodeJS에서 ElasticSearch API를 사용해 Index 내 Document 조회

 ```javascript
  var searchLog = function(body, callback){
          console.log("##body" + JSON.stringify(body));
          //파라미터 체크
          var index_title = body["index_title"];
          var query_param = body["query_param"];
          var start_date = body["startDate"]*1; //number타입 형변환
          var end_date = body["endDate"]*1;//number타입 형변환
          console.log("##"+index_title);
          if(typeof(index_title) === "undefined"
            || typeof(start_date) !== "number"
            || typeof(end_date) !== "number"
            || start_date>end_date){
                    throw {code:'OWL001', message:'Invalid Parameters'};
        }

          var query = query_param==""?{"match_all":{}}:{"query_string":{ "query": query_param}};

          var dsl = {
            "index": index_title,
            "size":conf["search-max-count"]?conf["search-max-count"]:500,
            //"size": body["w_size"],		//가져올 데이터 개수
            //"from": body["startData"],	//가져올 데이터 시작 번호 (from 부터 size 만큼 가져온다)
            "body": {
              "query": {
                "bool": {
                  "must": [
                           query,
                           { "range" :{ "@timestamp" : {"gte" : start_date,
                                          "lte" : end_date,
                                  "format" : "epoch_millis"}}}
                           ]
                }
              },
              "sort": [
                       { "@timestamp": "desc" }
                       ],
              "highlight": {
                "require_field_match": false,
                "fields": {
                  "*": {
                    "pre_tags": ["<mark>"],
                    "post_tags": ["</mark>"]
                  }
                }
              }
            }
          };

          client.search(dsl).then(function(result){     //Elastic Search 조회.
              var resultData = {};
              var data = [];
              var highlightFields = [];

              resultData.total = result.hits.total;

              for(var i=0;i<result.hits.hits.length;i++){
                var tempData = {};
                tempData._source = result.hits.hits[i]._source;
                tempData._time = result.hits.hits[i].sort[0];
                if(result.hits.hits[0].highlight) highlightFields = Object.keys(result.hits.hits[0].highlight); //highlight filed 추출.

                for(var key in result.hits.hits[i]._source){
                    var subData = result.hits.hits[i]._source[key];
                    //하위데이터가 JSON형식일 경우
                    //하위데이터의 key를 가지고 와 새로운 key를 만들어줌
                    if(subData && (subData.constructor === {}.constructor)){
                      for(var subKey in subData){
                        //highlight가 있는 필드라면 hight의 값을 넣음
                        if(result.hits.hits[i].highlight && result.hits.hits[i].highlight[key+"."+subKey]){
                            tempData._source[key+"."+subKey] = result.hits.hits[i].highlight[key+"."+subKey][0];
                            tempData[key+"."+subKey] = result.hits.hits[i].highlight[key+"."+subKey][0];
                        }
                        else tempData[key+"."+subKey] = subData[subKey];
                      }
                    }
                    else{
                      //highlight가 있는 필드라면 hight의 값을 넣음
                      if(result.hits.hits[i].highlight && result.hits.hits[i].highlight[key]){
                        // for(var hightlightKey in result.hits.hits[i].highlight[key] ){    //highlight
                          tempData._source[key] = result.hits.hits[i].highlight[key][0];
                          tempData[key] = result.hits.hits[i].highlight[key][0];
                        // }
                      }
                      else tempData[key] = subData;
                    }
                }
                console.log("##tempData",tempData);
                data.push(tempData);
              }
              resultData.data = data;
              resultData.highlight = highlightFields;

              //각 필드별 top5 꺼내기

              /*
              count = {
                field_name : {
                  "key1":0,
                  "key2":10,
                  ...
                },
                field_name2:{
                  "key1":0,
                  "key2":10,
                  ...
                },
                ...
              }
              */
              if(data.length!=0){
                var keys = Object.keys(data[0]);
                var count = {};
                for(var i=0;i<keys.length;i++){
                  var field_cnt = {};
                  for(var j=0;j<data.length;j++){
                    var row = data[j];
                    if(row[keys[i]]) field_cnt[row[keys[i]]] = field_cnt[row[keys[i]]]?++field_cnt[row[keys[i]]]:1;
                  }
                  //field_cnt 정렬
                  var sortData = [];
                  for(key in field_cnt){
                    sortData.push({"key":key,"value":field_cnt[key]});
                  }
                  //정렬
                  sortData.sort(function(a,b){
                    return(a.value<b.value)?-1:(a.value>b.value)?1:0;
                  }).reverse();
                  //console.log("sortData : " + JSON.stringify(sortData));
                  //정렬된 데이터를 다시 변환

                  var rankArr = [];
                  var len = sortData.length>5?5:sortData.length;
                  for(var j=0;j<len;j++){
                    var rankField = {};
                    rankField[sortData[j].key] = sortData[j].value;
                    rankArr.push(rankField);
                  }

                  count[keys[i]] = rankArr;
                }
              }
              //console.log("##count : " + JSON.stringify(count));
              resultData["count_by_field"] = count?count:0;
              resultData["max_count"] = conf["search-max-count"];
              callback(null, JSON.stringify(resultData));
          }, function(error){
              console.error("##ERROR.STACK : " + error.stack);
              var err = {};
              err.code = 'OWL001';
              err.message = 'Search Engine Error';
              err.stack = error.stack;
              callback("error", err);
          }
        ).catch(function(e){
          console.error("##CATCH : " + e.stack);
          var err = {};
          err.code = 'OWL999';
          err.message = e.message;
          err.stack = e.stack;
          callback("error", err);
        });
        }
 ```
