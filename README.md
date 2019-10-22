# private-data-doc

■ 참고 사이트 : https://miiingo.tistory.com/193?category=644184

■ Using Private Data in Fabric
권한이 부여된 조직의 블록 체인 네트워크에서 프라이빗 데이터를 저장 및 검색 할 수 있도록 컬렉션을 사용하는 방법을 가이드

Fabric으로 프라이빗 데이터를 정의, 구성 및 사용하는 단계별 순서
1. 컬렉션 정의 JSON 파일 만들기
2. 체인코드 API를 사용하여 프라이빗 데이터 읽기 및 쓰기
3. 컬렉션과 함께 체인코드 설치 및 인스턴스화
4. 프라이빗 데이터 저장
5. 승인된 피어로서 프라이빗 데이터 쿼리
6. 승인되지 않은 피어로서 프라이빗 데이터 쿼리
7. 프라이빗 데이터 삭제
8. 프라이빗 데이터에 인덱스 사용

< 1. Build a collection definition JSON file(컬렉션 정의 JSON 파일 만들기) >

1.1 컬렉션 정의는 속성 및 구성
name : 컬렉션의 이름
policy : 컬렉션 데이터를 유지할 수 있는 조직 피어를 정의
requiredPeerCount : 체인코드를 보증하기 위한 조건으로 프라이빗 데이터를 보급하는 데 필요한 피어의 수
maxPeerCount : 데이터 중복성을 위해 현재 피어 투 피어가 데이터를 배포하려고 시도하는 다른 피어의 수
blockToLive : 데이터를 블록 단위로 프라이빗 데이터베이스에 저장하는 기간
			  (프라이빗 데이터를 절대 삭제하지 않으려면 blockToLive 속성을 0으로 설정)
memberOnlyRead : true 값은 컬렉션 구성원 조직 중 하나에 속한 클라이언트만 프라이빗 데이터에 대한 읽기 액세스가 허용되도록 피어가 자동으로 적용함


[
  {
       "name": "collectionMarbles",
       "policy": "OR('Org1MSP.member', 'Org2MSP.member')",		//채널의 모든 구성원 (Org1 및 Org2)이 프라이빗 데이터베이스 프라이빗 데이터를 가질 수 있음
       "requiredPeerCount": 0,
       "maxPeerCount": 3,
       "blockToLive":1000000,
       "memberOnlyRead": true
  },

  {
       "name": "collectionMarblePrivateDetails",
       "policy": "OR('Org1MSP.member')",						//Org1의 구성원만 프라이빗 데이터베이스에 프라이빗 데이터를 가질 수 있음
       "requiredPeerCount": 0,
       "maxPeerCount": 3,
       "blockToLive":3,
       "memberOnlyRead": true
  }
]

< 2. Read and Write private data using chaincode APIs(체인코드 API를 사용하여 프라이빗 데이터 읽기 및 쓰기) >

2.1 두가지 데이터 엑세스 방법(marbles 프라이빗 데이터 샘플)
- name, color, size 및 owner가 채널의 모든 구성원에게 표시됩니다(Org1 및 Org2).
- price는 Org1의 구성원에게만 표시됩니다.
- 컬렉션 정의를 사용하여 프라이빗 데이터를 읽고 쓰는 작업은 GetPrivateData() 및 PutPrivateData()를 호출하여 수행
- 다이어드램은 참고 사이트 참조

// 1. Peers in Org1 and Org2 will have this private data in a side database
type marble struct {
  ObjectType string `json:"docType"`
  Name       string `json:"name"`
  Color      string `json:"color"`
  Size       int    `json:"size"`
  Owner      string `json:"owner"`
}

// 2. Only peers in Org1 will have this private data in a side database
type marblePrivateDetails struct {
  ObjectType string `json:"docType"`
  Name       string `json:"name"`
  Price      int    `json:"price"`
}

< 3. Reading collection data(컬렉션 데이터 읽기) >
- 체인 코드 API : GetPrivateData() 컬렉션 이름과 데이터 키 두개의 인수필요.
- readMarble : name, color, size 및 owner 속성 값을 쿼리
- readMarblePrivateDetails : price 속성 값을 쿼리

< 4. Writing private data(프라이빗 데이터 쓰기) >
- 체인 코드 API : PutPrivateData()를 사용
- collectionMarbles : name, color, size 및 owner를 저장
- collectionMarblePrivateDetails : price를 저장

// Code 예제
// ==== Create marble object, marshal to JSON, and save to state ====
      marble := &marble{
              ObjectType: "marble",
              Name:       marbleInput.Name,
              Color:      marbleInput.Color,
              Size:       marbleInput.Size,
              Owner:      marbleInput.Owner,
      }
      marbleJSONasBytes, err := json.Marshal(marble)
      if err != nil {
              return shim.Error(err.Error())
      }

      // === Save marble to state ===
      err = stub.PutPrivateData("collectionMarbles", marbleInput.Name, marbleJSONasBytes)
      if err != nil {
              return shim.Error(err.Error())
      }

      // ==== Create marble private details object with price, marshal to JSON, and save to state ====
      marblePrivateDetails := &marblePrivateDetails{
              ObjectType: "marblePrivateDetails",
              Name:       marbleInput.Name,
              Price:      marbleInput.Price,
      }
      marblePrivateDetailsBytes, err := json.Marshal(marblePrivateDetails)
      if err != nil {
              return shim.Error(err.Error())
      }
      err = stub.PutPrivateData("collectionMarblePrivateDetails", marbleInput.Name, marblePrivateDetailsBytes)
      if err != nil {
              return shim.Error(err.Error())
      }

< 5. Start the network(네트워크 시작) >

5.1 Fabric 이전 환경을 정리
cd fabric-samples/first-network
./byfn.sh down

5.1.1 Tutorial 이전 환경을 정리
docker rm -f $(docker ps -a | awk '($2 ~ /dev-peer.*.marblesp.*/) {print $1}')
docker rmi -f $(docker images | awk '($1 ~ /dev-peer.*.marblesp.*/) {print $3}')

5.2 Start the network : CouchDB로 BYFN 네트워크를 시작
$ ./byfn.sh up -c mychannel -s couchdb
$ docker ps -a

< 6. Install and instantiate chaincode with a collection(컬렉션과 함께 체인 코드 설치 및 인스턴스화) >
BYFN 네트워크에는 Org1 및 Org2라는 두 개의 조직이 있으며 각각 두 개의 피어가 있습니다. 따라서 체인 코드는 4 개의 피어에 설치되어야함.
- peer0.org1.example.com:7051
- peer1.org1.example.com:8051
- peer0.org2.example.com:9051
- peer1.org2.example.com:10051

6.1 Install chaincode on all peers(모든 피어에 체인 코드 설치)

6.1.1 peer chaincode install 명령을 사용하여 각 피어에 Marbles 체인코드 설치.
$ docker exec -it cli bash
=> (명령 프롬프트가 다음과 같이 바뀜) : root@81eac8493633:/opt/gopath/src/github.com/hyperledger/fabric/peer#

6.1.2 peer0.org1.example.com으로 Marbles 체인 코드를 설치
$ peer chaincode install -n marblesp -v 1.0 -p github.com/chaincode/marbles02_private/go/
=> (명령 프롬프트 결과) : install -> INFO 003 Installed remotely response:<status:200 payload:"OK" >

6.1.3 Org1의 두 번째 피어로 전환하고 체인 코드를 설치하십시오. 다음 전체 명령 블록을 복사하여 CLI 컨테이너에 붙여넣고 실행
$ export CORE_PEER_ADDRESS=peer1.org1.example.com:8051 
$ peer chaincode install -n marblesp -v 1.0 -p github.com/chaincode/marbles02_private/go/

6.1.4 Org2로 전환 후 다음 명령 블록을 그룹으로 복사하여 피어 컨테이너에 붙여넣고 실행
$ export CORE_PEER_LOCALMSPID=Org2MSP
$ export PEER0_ORG2_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
$ export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG2_CA
$ export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp

6.1.5 Org2에서 활성 피어를 첫 번째 피어로 전환하고 체인 코드를 설치
$ export CORE_PEER_ADDRESS=peer0.org2.example.com:9051
$ peer chaincode install -n marblesp -v 1.0 -p github.com/chaincode/marbles02_private/go/

6.1.6 활성 피어를 Org2의 두 번째 피어로 전환하고 체인 코드를 설치
$ export CORE_PEER_ADDRESS=peer1.org2.example.com:10051
$ peer chaincode install -n marblesp -v 1.0 -p github.com/chaincode/marbles02_private/go/

6.2 Instantiate the chaincode on the channel(채널에서 체인 코드를 인스턴스화)
다음 명령을 실행하여 BYFN의 채널인 mychannel에서 marbles 프라이빗 데이터 체인 코드를 인스턴스화
$ export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
$ peer chaincode instantiate -o orderer.example.com:7050 --tls --cafile $ORDERER_CA -C mychannel -n marblesp -v 1.0 -c '{"Args":["init"]}' -P "OR('Org1MSP.member','Org2MSP.member')" --collections-config  $GOPATH/src/github.com/chaincode/marbles02_private/collections_config.json
(주의 : --collections-config 플래그의 값을 지정시 절대 경로를 지정해야합니다. 예를 : --collections-config  $GOPATH/src/github.com/chaincode/marbles02_private/collections_config.json )

=> (명령 프롬프트 결과) : [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
						  [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
						  
< 7. Store private data(프라이빗 데이터 저장) >
Org1의 구성원으로 활동하면서 Org1의 피어로 다시 전환하여 marbles 추가 요청(request)을 제출(submit)하는 흐름.

$ export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
$ export CORE_PEER_LOCALMSPID=Org1MSP
$ export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
$ export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
$ export PEER0_ORG1_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt

$ export MARBLE=$(echo -n "{\"name\":\"marble1\",\"color\":\"blue\",\"size\":35,\"owner\":\"tom\",\"price\":99}" | base64 | tr -d \\n)
$ peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n marblesp -c '{"Args":["initMarble"]}'  --transient "{\"marble\":\"$MARBLE\"}"
=> (명령 프롬프트 결과) : [chaincodeCmd] chaincodeInvokeOrQuery->INFO 001 Chaincode invoke successful. result: status:200

< 8. Query the private data as an authorized peer(승인된 피어로서 프라이빗 데이터 쿼리) >
collectionMarbles & readMarble API 경우 : Org1 및 Org2의 모든 구성원(member)이 name, color, size, owner 프라이빗 데이터를 Side database에 저장
collectionMarblePrivateDetails & readMarblePrivateDetails API 경우 : Org1의 피어만 Sida database에 price 프라이빗 데이터를 저장

8.1 첫 번째 query내용 : collectionMarbles를 인수로 전달하는 readMarble 함수호출(코드예제)

// ===============================================
// readMarble - read a marble from chaincode state
// ===============================================
func (t *SimpleChaincode) readMarble(stub shim.ChaincodeStubInterface, args []string) pb.Response {
     var name, jsonResp string
     var err error
     if len(args) != 1 {
             return shim.Error("Incorrect number of arguments. Expecting name of the marble to query")
     }

     name = args[0]
     valAsbytes, err := stub.GetPrivateData("collectionMarbles", name) //get the marble from chaincode state

     if err != nil {
             jsonResp = "{\"Error\":\"Failed to get state for " + name + "\"}"
             return shim.Error(jsonResp)
     } else if valAsbytes == nil {
             jsonResp = "{\"Error\":\"Marble does not exist: " + name + "\"}"
             return shim.Error(jsonResp)
     }

     return shim.Success(valAsbytes)
}

8.1.2 두 번째 query내용 : collectionMarblePrivateDetails를 인수로 전달하는 readMarblePrivateDetails 함수호출(코드예제)

// ==============================================================================
// readMarblePrivateDetails - read a marble private details from chaincode state
// ==============================================================================
func (t *SimpleChaincode) readMarblePrivateDetails(stub shim.ChaincodeStubInterface, args []string) pb.Response {
     var name, jsonResp string
     var err error

     if len(args) != 1 {
             return shim.Error("Incorrect number of arguments. Expecting name of the marble to query")
     }

     name = args[0]
     valAsbytes, err := stub.GetPrivateData("collectionMarblePrivateDetails", name) //get the marble private details from chaincode state

     if err != nil {
             jsonResp = "{\"Error\":\"Failed to get private details for " + name + ": " + err.Error() + "\"}"
             return shim.Error(jsonResp)
     } else if valAsbytes == nil {
             jsonResp = "{\"Error\":\"Marble private details does not exist: " + name + "\"}"
             return shim.Error(jsonResp)
     }
     return shim.Success(valAsbytes)
}

8.2 Org1의 구성원으로 marble1의 name, color, size 및 owner 프라이빗 데이터를 쿼리
$ peer chaincode query -C mychannel -n marblesp -c '{"Args":["readMarble","marble1"]}'
=> (명령 프롬프트 결과) : {"color":"blue","docType":"marble","name":"marble1","owner":"tom","size":35}

8.2.1 Org1의 구성원으로 marble1의 price 프라이빗 데이터를 쿼리
$ peer chaincode query -C mychannel -n marblesp -c '{"Args":["readMarblePrivateDetails","marble1"]}'
=> (명령 프롬프트 결과) : {"docType":"marblePrivateDetails","name":"marble1","price":99}

< 9. Query the private data as an unauthorized peer(승인되지 않은 피어로서 프라이빗 데이터 쿼리) >
marbles의 price 프라이빗 데이터가 없는 Org2의 구성원으로 전환 후 프라이빗 데이터의 두 세트를 모두 쿼리 수행하는 학습

9.1 Switch to a peer in Org2(Org2의 피어로 전환)
$ export CORE_PEER_ADDRESS=peer0.org2.example.com:9051
$ export CORE_PEER_LOCALMSPID=Org2MSP
$ export PEER0_ORG2_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
$ export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG2_CA
$ export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp

9.2 Query private data Org2 is authorized to(Org2가 승인된 프라이빗 데이터 쿼리)
$ peer chaincode query -C mychannel -n marblesp -c '{"Args":["readMarble","marble1"]}'
=> (명령 프롬프트 결과) : {"docType":"marble","name":"marble1","color":"blue","size":35,"owner":"tom"}

9.3 Query private data Org2 is not authorized to(Org2가 승인되지 않은 프라이빗 데이터 쿼리)
$ peer chaincode query -C mychannel -n marblesp -c '{"Args":["readMarblePrivateDetails","marble1"]}'
=> (명령 프롬프트 결과) : {"Error":"Failed to get private details for marble1: GET_STATE failed:~~~~

< 10. Purge Private Data(프라이빗 데이터 삭제) >
전제 시나리오 : 개인 정보 또는 기밀 정보(예: 가격 데이터)를 비롯한 프라이빗 데이터가 있을 수 있으며 거래 당사자는 채널의 다른 조직에 공개하기를 원하지 않음.
				제한된 수명을 가지며 컬렉션 정의에서 blockToLive 속성을 사용하여 지정된 블록 수만큼 블록 체인에서 변경되지 않은 상태로 제거될 수 있음.
시스템 상황 : collectionMarblePrivateDetails는 blockToLive 속성 값을 3으로 정의됨, 세 블록 동안만 Side database에 저장하고 그 이후에는 제거됨을 의미함.
			  네 번째 거래(세 번째 marble 이동)가 끝나면 가격(price) 프라이빗 데이터가 삭제되었는지 확인합니다.
			  Org1의 peer0으로 다시 전환 후 아래 명령어들을 실행해서 확인합니다.
			  
10.1 Org1의 peer0으로 다시 전환 명령어
$ export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
$ export CORE_PEER_LOCALMSPID=Org1MSP
$ export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
$ export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
$ export PEER0_ORG1_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt

10.2 새 터미널 창을 열고 다음 명령을 실행하여 이 피어에 대한 프라이빗 데이터 로그를 확인
$ docker logs peer0.org1.example.com 2>&1 | grep -i -a -E 'private|pvt|privdata'
=> (명령 프롬프트 결과) : [mychannel] Committed block [6] with 1 transaction(s) in 224ms	
                          (가장 높은 블럭높이 숫자[6] 확인)

10.3 피어 컨테이너로 돌아가서 다음 명령을 실행하여 marble1의 price 데이터를 쿼리 (쿼리는 거래되는 데이터가 없으므로 원장에 새 트랜잭션을 생성하지 않습니다.)
$ peer chaincode query -C mychannel -n marblesp -c '{"Args":["readMarblePrivateDetails","marble1"]}'
=> (명령 프롬프트 결과) : {"docType":"marblePrivateDetails","name":"marble1","price":99} 	
                          (price 데이터는 여전히 프라이빗 데이터 원장에 남아있습니다.)

10.4 다음 명령을 실행하여 새로운 marble2를 만듭니다. 이 트랜잭션은 체인에 새로운 블록을 생성.
$ export MARBLE=$(echo -n "{\"name\":\"marble2\",\"color\":\"blue\",\"size\":35,\"owner\":\"tom\",\"price\":99}" | base64 | tr -d \\n)
$ peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n marblesp -c '{"Args":["initMarble"]}' --transient "{\"marble\":\"$MARBLE\"}"

10.4.1 터미널 창으로 다시 전환하고 이 피어의 프라이빗 데이터 로그를 다시 확인합니다. 블록 높이가 1씩 증가해야 합니다.(6 => 7)
$ docker logs peer0.org1.example.com 2>&1 | grep -i -a -E 'private|pvt|privdata'

10.4.2 피어 컨테이너로 돌아가서 다음 명령을 실행하여 marble1의 price 데이터를 다시 쿼리합니다.
$ peer chaincode query -C mychannel -n marblesp -c '{"Args":["readMarblePrivateDetails","marble1"]}'
=> (명령 프롬프트 결과) : {"docType":"marblePrivateDetails","name":"marble1","price":99}
						  (프라이빗 데이터가 삭제되지 않았으므로 결과는 이전 쿼리와 동일합니다.)

10.5 다음 명령을 실행하여 marble2를 "joe"에게로 이동합니다. 이 트랜잭션은 체인에 두 번째 새로운 블록을 추가합니다.
$ export MARBLE_OWNER=$(echo -n "{\"name\":\"marble2\",\"owner\":\"joe\"}" | base64 | tr -d \\n)
$ peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n marblesp -c '{"Args":["transferMarble"]}' --transient "{\"marble_owner\":\"$MARBLE_OWNER\"}"

10.5.1 터미널 창으로 다시 전환하고 이 피어의 프라이빗 데이터 로그를 다시 확인합니다. 블록 높이가 1씩 증가해야 합니다. (7 => 8)
$ docker logs peer0.org1.example.com 2>&1 | grep -i -a -E 'private|pvt|privdata'

10.5.2 피어 컨테이너로 돌아가서 다음 명령을 실행하여 marble1의 price 데이터를 다시 쿼리합니다.
$ peer chaincode query -C mychannel -n marblesp -c '{"Args":["readMarblePrivateDetails","marble1"]}'
=> (명령 프롬프트 결과) : {"docType":"marblePrivateDetails","name":"marble1","price":99}
                          (여전히 price 데이터가 보입니다.)

10.6 다음 명령을 실행하여 marble2를 "tom"에게로 이동합니다. 이 트랜잭션은 체인에 세 번째 새로운 블록을 추가합니다.
$ export MARBLE_OWNER=$(echo -n "{\"name\":\"marble2\",\"owner\":\"tom\"}" | base64 | tr -d \\n)
$ peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n marblesp -c '{"Args":["transferMarble"]}' --transient "{\"marble_owner\":\"$MARBLE_OWNER\"}" 

10.6.1 터미널 창으로 다시 전환하고 이 피어의 프라이빗 데이터 로그를 다시 확인합니다. 블록 높이가 1씩 증가해야 합니다. (8 => 9)
$ docker logs peer0.org1.example.com 2>&1 | grep -i -a -E 'private|pvt|privdata'

10.6.2 피어 컨테이너로 돌아가서 다음 명령을 실행하여 marble1의 price 데이터를 다시 쿼리합니다.
$ peer chaincode query -C mychannel -n marblesp -c '{"Args":["readMarblePrivateDetails","marble1"]}'
=> (명령 프롬프트 결과) : {"docType":"marblePrivateDetails","name":"marble1","price":99}
                          (여전히 price 데이터가 보입니다.)
						  
10.7 마지막으로 다음 명령을 실행하여 marble2를 "jerry"에게로 이동합니다. 이 트랜잭션은 체인에 네 번째 새로운 블록을 추가합니다. (price 프라이빗 데이터는 이 거래 후 제거되어야 합니다.)
$ export MARBLE_OWNER=$(echo -n "{\"name\":\"marble2\",\"owner\":\"jerry\"}" | base64 | tr -d \\n)
$ peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n marblesp -c '{"Args":["transferMarble"]}' --transient "{\"marble_owner\":\"$MARBLE_OWNER\"}"

10.7.1 터미널 창으로 다시 전환하고 이 피어의 프라이빗 데이터 로그를 다시 확인합니다. 블록 높이가 1씩 증가해야 합니다. (9 => 10)
$ docker logs peer0.org1.example.com 2>&1 | grep -i -a -E 'private|pvt|privdata'

10.7.2 피어 컨테이너로 돌아가서 다음 명령을 실행하여 marble1의 price 데이터를 다시 쿼리합니다.
$ peer chaincode query -C mychannel -n marblesp -c '{"Args":["readMarblePrivateDetails","marble1"]}'
=> (명령 프롬프트 결과) : Error: endorsement failure during query. response: status:500 message:"{\"Error\":\"Marble private details does not exist: marble1\"}"
						 (price 데이터가 제거되었으므로 더 이상 볼 수 없습니다.)

< 참고 명령어 : Clean Up >
cd /fabric-samples/first-network
./byfn.sh down
docker rm $(docker ps -aq)
docker rmi $(docker images dev-* -q)
docker network prune
< 강제로 삭제시 -f >
docker rm -f ID

