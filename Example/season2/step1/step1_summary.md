# RxSwift 4시간만에 끝내기 (step1)

# Observable

---

### 문제점

- Load 버튼을 누르면 Api 통신을해 json을 받아와 화면에 뿌린다
- 하지만 데이터 받아오는 동안 타이머랑 버튼이 잠깐 멈춘다

```swift
@IBAction func onLoad() {
  editView.text = ""
  setVisibleWithAnimation(activityIndicator, true)

  let url = URL(string: MEMBER_LIST_URL)!
  let data = try! Data(contentsOf: url)
  let json = String(data: data, encoding: .utf8)
  self.editView.text = json

  self.setVisibleWithAnimation(self.activityIndicator, false)
}
```



### 왜 멈추냐?

- 다운로드 받는 작업이 동기화로 작업되고 있다

- json을 다운로드 받는 것과 라벨이 잠시 멈추고 다시 부활하는 것을 따로 처리해야한다

- `비동기`로 바꾸어 주면 된다?

   ```swift
   @IBAction func onLoad() {
     editView.text = ""
     setVisibleWithAnimation(activityIndicator, true)
   
     DispatchQueue.global().async{
     let url = URL(string: MEMBER_LIST_URL)!
     let data = try! Data(contentsOf: url)
     let json = String(data: data, encoding: .utf8)
     
       DispatchQueue.main.async{
         self.editView.text = json
         self.setVisibleWithAnimation(self.activityIndicator, false)
       }
     }
   }
   ```

- 다른 Thread를 사용해서 다른 작업과 현재 작업을 동시에 수행 결과를 비동기적으로 받아서 표현

### 한번에 끝내보자

```swift
func downloadJson(_ url: String, _ completion: @escaping (String?) -> Void) -> String?{
  DispatchQueue.global().async{
    let url = URL(string: url)!
    let data = try! Data(contentsOf: url)
    let json = String(data: data, encoding: .utf8)
    DispatchQueue.main.asycn{
    	completion(json)
    }
  }
}

@IBAction func onLoad() {
  editView.text = ""
  setVisibleWithAnimation(activityIndicator, true)

  downloadJson(MEMBER_LIST_URL) { json in
    self.editView.text = json
    self.setVisibleWithAnimation(self.activityIndicator, false)
  }
}
```

- 여태까지 다른 쓰레드에서 코드를 처리할 떄는 이런 방식으로 사용해 왔다
- 만약 json을 겁나 많이 받아야한다....? 콜백지옥....!

```swift
@IBAction func onLoad() {
  editView.text = ""
  setVisibleWithAnimation(activityIndicator, true)

  self.downloadJson(MEMBER_LIST_URL) { json in
    self.editView.text = json
    self.setVisibleWithAnimation(self.activityIndicator, false)
    
    self.downloadJson(MEMBER_LIST_URL) { json in
      self.editView.text = json
      self.setVisibleWithAnimation(self.activityIndicator, false)
			
      self.downloadJson(MEMBER_LIST_URL) { json in
          self.editView.text = json
          self.setVisibleWithAnimation(self.activityIndicator, false)
        }
    }                                  
  }
}
```

- completion을 사용하지 말고 return을 사용하면 더 편할텐데...!

### RxSwift 나온 배경

- 비동기로 나오는 데이터를 어떻게 return값으로 받을까?
- 이런걸 해결해주는 유틸리티가 나오게 됌

```swift
func downloadJson(_ url: String ) -> 나중에생기는데이터<String?>{
  return 나중에생기는데이터() { f in
    DispatchQueue.global().async{
    let url = URL(string: url)!
    let data = try! Data(contentsOf: url)
    let json = String(data: data, encoding: .utf8)
    DispatchQueue.main.asycn{
    	completion(json)
    	}
  	}
  }
}


@IBAction func onLoad() {
  editView.text = ""
  setVisibleWithAnimation(activityIndicator, true)

  let json:나중에생기는데이터<String> = downloadJson(MEMBER_LIST_URL)
  
  json.나중에오면{ json in
    self.editView.text = json
    self.setVisibleWithAnimation(self.activityIndicator, false)
  }
}
```



### 나중에 생기는 데이터?

```swift
class 나중에생기는데이터<T>{
  private let task: (@escaping (T)-> Void) -> Void
 
  // 생성시 클로저를 받아서 가지고 있다가
  init(task: @escaping (@escaping (T) -> Void) -> Void){
    self.task = task
  }
  
  // 나중에오면 이라는 함수가 실행할 때 전달받은 클로져 실행 후 지금 들어온 클로져 전달
  func 나중에오면(_ f: @escaping (T) -> Void){
    task(f)
  }
}
```

- 원하는 동작을 그대로 처리하면서 비동기적으로 생기는 데이터를 completion을 쓰지않고 리턴한다

### 이러한 유틸리티들,,,?

- PromiseKit
- Blot
- RxSwift
   - 나중에생기는 데이터 -> Observable
   - 만들때 -> create()
   - 전달 completion -> onNext()
   - 나중에 오면 -> subscribe(evnet :)
      - event는 전달한 onNext()로 전달한 이벤트가 온다
      - event는 .next, .completed, .error가 있다

### RxSwift의 용도

- 비동기로 생기는 결과 값을 completion과 같은 클로져 전달하는게 아니라 리턴값으로 전달하기 위해 만들어진 유틸리티

### Disposable

- dispose라는 함수를 실행
- 진행중인 작업을을 끝내지 않았어도 취소 시킬 수 있다

### 클로져 내의 self

- 순환참조 유발
- [weak self], self? 로 사용
- 클로저 내에서 self를 캡쳐하여 레퍼런스 카운트가 증가하기 때문에 발생
- 클로저가 생성되면서 rc가 증가 했기에 클로저가 사라지면 rc가 감소
- 이때 subscribe의 rc는 completed 나 error때 사라짐
- f.onCompleted() 을 선언함으로 순환 참조 문제 해결 // 더 알아보기
- dispose 시켜도 순환참조 없어짐

### RxSwift 사용방법?

1. 비동기로 생기는 데이터를 Observable로 감싸서 리턴하는 방법 -> dwonloadJson 내부 Observable.create()
2. Observable로 오는 데이터를 받아서 처리하는 방법 -> 버튼 내부 .subscribe

### 제대로 만들면? Observable을 제대로 만드는법

```swift
func downloadJson(_ url: String) -> Observable<String>{
        // 1. 비동기로 생기는 데이터를 Observable로 감싸서 리턴하는 타입
        return Observable.create() { emitter in
            let url = URL(string: url)!
            let task = URLSession.shared.dataTask(with: url){ (data, _ , err) in
                guard err == nil else {
                    emitter.onError(err!)
                    return
                }
                
                if let dat = data, let json = String(data: dat, encoding: .utf8){
                    // next로 json전달
                    emitter.onNext(json)
                }
                
                emitter.onCompleted() // 끝났다
            }
            
            task.resume()
            return Disposables.create(){
                // 중간에 캔슬하면
                task.cancel()
            }

        }
    }


// 변수로 할당하여 구현하기
@IBAction func onLoad() {
  editView.text = ""
  setVisibleWithAnimation(activityIndicator, true)

  let observable = downloadJson(MEMBER_LIST_URL)
        
  
  let dispose = observable.subscribe{ event in
           switch event {
           case let .next(json):
              DispatchQueue.main.async {
                  self.editView.text = json
                  self.setVisibleWithAnimation(self.activityIndicator, false)
              }
           case .completed:
              break
           case .error:
              break
           }
   		}
}
```

- 이렇게 생긴 Observable은 subscribe에서 Dispose가 불렸을 떄 task.cancel() 을 수행한다
- 데이터가 준비가 되면 onNext()를 보내지만 준비가 안됐을 시 그냥 onCompleted()가 불린다
- subscribe 클로져 실행시 순환참조가 발생하게 되는데 onError / onCompleted / dispose 셋중하나 불리게 되면 클로져가 끝나서 순환참조가 사라진다

### 구성

- Observable을 `create` 한다
- emitter에 적절한 시점에` error`호출 , next로 데이터 전달
- `completed`로 종료 (Observable의 끝에 선언)
- 취소 해야할 때 해야하는 행동은 `Disposable.create()`을 return할때 기술한다

### Observable의 생명주기

1. Create
   - create됐다고 무조건 observable이 동작하는 것이 아니다
2. Subscribe
   - 동작은 이때한다
   - 동작이 시작되면 complete이나 error나 dispspoed 로인해 끝나고 
   - 끝난 observable은 다시 수행하지 못한다
3. onNext 
4. onCompleted // onError
5. Disposed

```swift
let ob = downloadJson(MEMBER_LIST_URL) // create
ob.subscrib{
  // 이때 동작!
}
```

### 다 됐을까?

- 위 상황으로 코딩을 해서 다시 subscribe 하는 곳으로 돌아온다면 보라색 에러메세지를 마주하게된다
- downloadJson을 동작하는 쓰레드는 메인 쓰레드가 아니기 때문에 다시 돌아와도 후처리를 downloadJson을 동작하던 쓰레드가 처리를 하게 된다 
- 그런데 onLoad 함수에서 ui를 바꾸는 작업을 진행하고 있다 !! -> error

```swift
@IBAction func onLoad() {
  editView.text = ""
  setVisibleWithAnimation(activityIndicator, true)

  downloadJson(MEMBER_LIST_URL)
    .subscribe{ event in
               switch event {
                 case let .next(json):
                   DispatchQueue.main.async {
                     self.editView.text = json
                     self.setVisibleWithAnimation(self.activityIndicator, false)
                   }
                 case .completed:
                 	break
                 case .error:
                 	break
               }
            }
}
```









