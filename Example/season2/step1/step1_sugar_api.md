# RxSwift 4시간만에 끝내기 (step1)

# Sugar Api

---

### RxSwift의 용도

- 비동기적으로 생기는 데이터를 리턴값으로 전달하기 위해 나중에 생기는 데이터(Observable)라는 class로 감싸서 전달
- 사용법이 길다? 귀찮다!!

### SugarApi

- 귀찮은 것들을 없애주는 api가 존재한다

### Just, from

```swift
func sendHelloWorld() -> Observable<String>{
    return Observable.create{ emitter in
        emitter.onNext("Hello World")
        emitter.onCompleted()
        return Disposables.create()
    }
}
```

- "Hello world"하나 보내는데 너무 길다! -> 귀찮다

```swift
func sendHelloWorld() -> Observable<String?>{
    return Observable.just("hello world")
}
// Optiona("Hello")
```

- Just로 한번에 정리, 4줄을 1줄로!
- Dispose됐을 때 특별히 아무 일도 일어나지 않을 때

```swift
// "hello", "world" 이렇게 따로보내고 싶을땐?
func sendHelloWorld() -> Observable<[String?]>{
    return Observable.just(["hello", "world"])
}
// [Optiona("Hello"), Optiona("Hello")]

// from을쓰면?
func sendHelloWorld() -> Observable<String?>{
    return Observable.from(["hello", "world"])
}
// Optiona("Hello")
// Optiona("wrold") 두번 출력 됌
```

- Just나 from을 쓰면 한줄로 표현 가능



### subscribe 한줄로 표현

- 한줄로 표현가능

```swift
func onLoad(){
  downloadJson(MEMBER_LIST_URL)
  	.subscribe(onNext: { print($0) },
               onCompleted: { print("Com")},
               onError: {err in print(err)})
}
```

- 3개 중에 선택적으로 정의 가능

### observeOn

```swift
@IBAction func onLoad() {
  editView.text = ""
  setVisibleWithAnimation(activityIndicator, true)

  downloadJson(MEMBER_LIST_URL)
    .subscribe(onNext: { json in
      DispatchQueue.main.async {
        self.editView.text = json
        self.setVisibleWithAnimation(self.activityIndicator, false)
      }
    })
}


@IBAction func onLoad() {
  editView.text = ""
  setVisibleWithAnimation(activityIndicator, true)

  downloadJson(MEMBER_LIST_URL)
  	.observeOn(MainScheduler.instance) // sugar API
    .subscribe(onNext:{ json in
      	self.editView.text = json
        self.setVisibleWithAnimation(self.activityIndicator, false)
    	}
    )
}
```

- DispatchQueue를 선언 안해줄 수도 있음



### Operator

- 데이터가 밑으로 내려가면서 중간에 바꾸는 sugar를 Operator 라고한다
- Observable과 subscribe 사이에 데이터를 바꿔서 가공해주는 것
- 엄청 많음
- map, filter,.. 등등





