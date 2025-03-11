# MapKit - iOS

Map 관련 연습

## 경로 기록

<p align="center">
  <table style="width:100%; text-align:center; border-spacing:20px;">
    <tr>
      <td style="text-align:center; vertical-align:middle;">
        <p align="center">
        <img src="https://github.com/BOLTB0X/MapKit-iOS/blob/main/history/%EA%B8%B0%EB%A1%9D%EA%B2%BD%EB%A1%9C%EB%82%A8%EA%B8%B0%EA%B8%B03.png?raw=true" 
             alt="1" 
             style="width:200px; height:400px; object-fit:contain;"/>
        </p>
      </td>
      <td style="text-align:center; vertical-align:middle;">
        <p align="center">
        <img src="https://github.com/BOLTB0X/MapKit-iOS/blob/main/history/%EC%A0%95%EC%A7%80+%EC%9E%AC%EA%B0%9C.gif?raw=true" 
             alt="2" 
             style="width:200px; height:400px; object-fit:contain;"/>
        </p>
      </td>
    </tr>
    <tr>
      <td style="text-align:center; font-size:14px; font-weight:bold;">
      <p align="center">
        경로 남기기
      </p>
      </td>
      <td style="text-align:center; font-size:14px; font-weight:bold;">
      <p align="center">
        정지 / 재개
      </p>
      </td>
    </tr>
  </table>
</p>

<details>
<summary> 경로 뷰 반영 </summary>

`MKDirections` 와 `MKPolylineRenderer` 을 이용 시 **MapKit** 에서 인식되는 경로만 선으로 이어버림

그러므로 사용자의 위치 데이터를 직접 받아서 선(`MKPolyline`)으로 이어서 지도에 표시하는 방식

```swift
struct PathMapView: UIViewRepresentable {
    // MARK: - 프로퍼티s
    var userLocations: [CLLocationCoordinate2D]
    var isRecording: Bool
    var timerState: TimerState

    // MARK: - makeUIView
    // 뷰가 그려질 때 호출
    func makeUIView(context: Context) -> MKMapView {
        let mapView = MKMapView()
        mapView.showsUserLocation = true
        mapView.setUserTrackingMode(.follow, animated: true)
        mapView.isZoomEnabled = true
        mapView.delegate = context.coordinator
        return mapView
    }

    // MARK: - updateUIView
    // 변경 시 업데이트 뷰
    func updateUIView(_ uiView: MKMapView, context: Context) {
        uiView.removeOverlays(uiView.overlays)

        if timerState != .clear && userLocations.count > 1 {
            let polyline = MKPolyline(coordinates: userLocations, count: userLocations.count)
            uiView.addOverlay(polyline)

            let region = MKCoordinateRegion(center: userLocations.last!, span: MKCoordinateSpan(latitudeDelta: 0.005, longitudeDelta: 0.005))
            uiView.setRegion(region, animated: true)
        }
    }

    // MARK: - makeCoordinator
    // 좌표상에 MKPolylineRenderer
    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }
}
```

</details>

<details>
<summary> 위치 권한 요청 </summary>

```swift
// MARK: - locationManager
// 위치
// ...
func locationManager(_ manager: CLLocationManager, didChangeAuthorization status: CLAuthorizationStatus) {
    switch status {
    case .authorizedAlways, .authorizedWhenInUse:
        print("GPS 권한 설정됨")
        startUpdatingLocation()
        locationManager.allowsBackgroundLocationUpdates = true
    case .restricted, .notDetermined:
          print("GPS 권한 설정되지 않음")
          getLocationUsagePermission()
      case .denied:
          print("GPS 권한 요청 거부됨")
          getLocationUsagePermission()
      default:
          print("GPS: Default")
      }
}

func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
    guard let location = locations.last else { return }
    print("위치 변경 감지, 위치 업데이트")
    currentLatitude = location.coordinate.latitude
    currentLongitude = location.coordinate.longitude

    convertLocationToAddress(location: location)

    //print(currentLatitude, currentLongitude)

    if timerState == .play {
        isPossibleRecord(location.coordinate)
    }
}


// MARK: - CLLocationManagerDelegate 관련
// ...
// MARK: - getLocationUsagePermission
func getLocationUsagePermission() {
    locationManager.requestWhenInUseAuthorization()
}

// MARK: - startUpdatingLocation
func startUpdatingLocation() {
    locationManager.startUpdatingLocation()
}

// MARK: - startUpdatingLocation
func stopUpdatingLocation() {
    locationManager.stopUpdatingLocation()
}
```

</details>

<details>
<summary> 기록 관련 뷰모델 </summary>

`locationManager: CLLocationManager` 으로 직접 위도/경도를 10m 씩 필터링

```swift
class PathRecordViewModel: NSObject, ObservableObject, CLLocationManagerDelegate {
  // 생략
  // ...
  private lazy var locationManager: CLLocationManager = {
        let manager = CLLocationManager()
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.distanceFilter = 10
        manager.startUpdatingLocation()
        manager.delegate = self
        return manager
  }()

  // ...

  // MARK: - isPossibleRecord
  // 기록할지 말지
  private func isPossibleRecord(_ location: CLLocationCoordinate2D) {
        if userLocations.isEmpty || distanceCoordinates(userLocations.last!, location) > 10.0 {
            userLocations.append(location)
        }
  }

}
```

`timerState` 로 현재 경로를 저장여부 판단, 거리 변환 관련 메서드들

```swift
class PathRecordViewModel: NSObject, ObservableObject, CLLocationManagerDelegate {
  // 생략
  // ...

  @Published var timerState: TimerState = .clear {
        didSet {
            switch timerState {
            case .clear:
                clearTimer()
            case .pause:
                pauseTimer()
            case .play:
                startTimer()
            }
        }
  }
  // ...

// MARK: - isPossibleRecord
// 기록할지 말지
private func isPossibleRecord(_ location: CLLocationCoordinate2D) {
    if userLocations.isEmpty || distanceCoordinates(userLocations.last!, location) > 10.0 {
            userLocations.append(location)
        }
  }

  // MARK: - distanceCoordinates
  // 거리 계산
  private func distanceCoordinates(_ coordinate1: CLLocationCoordinate2D, _ coordinate2: CLLocationCoordinate2D) -> CLLocationDistance {
      let location1 = CLLocation(latitude: coordinate1.latitude, longitude: coordinate1.longitude)
      let location2 = CLLocation(latitude: coordinate2.latitude, longitude: coordinate2.longitude)

      return location1.distance(from: location2)
  }

  // MARK: - calculateTotalDist
  // 위치 경도로 거리계산
  // 들어온 순서로 차잇값을 구한 뒤 distance 이용
  private func calculateTotalDist() {
      var totalDistance: CLLocationDistance = 0.0

      if userLocations.count <= 1 { return }

      for i in 0..<(userLocations.count - 1) {
          let location1 = CLLocation(latitude: userLocations[i].latitude, longitude: userLocations[i].longitude)
          let location2 = CLLocation(latitude: userLocations[i + 1].latitude, longitude: userLocations[i + 1].longitude)

          totalDistance += location1.distance(from: location2)
      }

      recordingMeter = "이동 거리 \(String(format: "%.2f", totalDistance))미터"
  }
}
```

</details>

- [PathRecord 관련 코드](https://github.com/BOLTB0X/MapKit-iOS/tree/main/MapKitSwiftUi02/MapKitSwiftUI/PathRecord)

## 백그라운드 상태

<p align="center">
  <table style="width:100%; text-align:center; border-spacing:20px;">
    <tr>
      <td style="text-align:center; vertical-align:middle;">
        <p align="center">
        <img src="https://github.com/BOLTB0X/MapKit-iOS/blob/main/history/%EB%B0%B1%EA%B7%B8%EB%9D%BC%EC%9A%B4%EB%93%9C%20%EC%83%81%ED%83%9C05.gif?raw=true" 
             alt="1" 
             style="width:200px; height:400px; object-fit:contain;"/>
        </p>
      </td>
      <td style="text-align:center; vertical-align:middle;">
        <p align="center">
        <img src="https://github.com/BOLTB0X/MapKit-iOS/blob/main/history/%EB%B0%B1%EA%B7%B8%EB%9D%BC%EC%9A%B4%EB%93%9C%EC%83%81%ED%83%9C_%EC%A3%BC%EB%A8%B8%EB%8B%88%EC%86%8D+30%EB%AF%B8%ED%84%B0%EA%B1%B8%EC%9D%8C%ED%9B%84%EC%8A%A4%EC%83%B7.png?raw=true" 
             alt="2" 
             style="width:200px; height:400px; object-fit:contain;"/>
        </p>
      </td>
    </tr>
    <tr>
      <td style="text-align:center; font-size:14px; font-weight:bold;">
      <p align="center">
        백그라운드 상태
      </p>
      </td>
      <td style="text-align:center; font-size:14px; font-weight:bold;">
      <p align="center">
        백그라운드상태 로 30m 걸음 후
      </p>
      </td>
    </tr>
  </table>
</p>

## Firebase 연동

<p align="center">
  <table style="width:100%; text-align:center; border-spacing:20px;">
    <tr>
      <td style="text-align:center; vertical-align:middle;">
        <p align="center">
        <img src="https://github.com/BOLTB0X/MapKit-iOS/blob/main/history/05_%EC%8B%A4%EA%B8%B0%EA%B8%B0%ED%85%8C%EC%8A%A4%ED%8A%B8_firebaseDB%EC%97%B0%EB%8F%99_2.gif?raw=true" 
             alt="1" 
             style="width:200px; height:400px; object-fit:contain;"/>
        </p>
      </td>
      <td style="text-align:center; vertical-align:middle;">
        <p align="center">
        <img src="https://github.com/BOLTB0X/MapKit-iOS/blob/main/history/05_%EC%8B%A4%EA%B8%B0%EA%B8%B0%ED%85%8C%EC%8A%A4%ED%8A%B8_firebaseDB%EC%97%B0%EB%8F%99_%ED%8C%8C%EC%9D%B4%EC%96%B4%EB%B2%A0%EC%9D%B4%EC%8A%A4%20%EC%8A%A4%EC%83%B7.png?raw=true" 
             alt="2" 
             style="width:200px; height:400px; object-fit:contain;"/>
        </p>
      </td>
    </tr>
    <tr>
      <td style="text-align:center; font-size:14px; font-weight:bold;">
      <p align="center">
        Firebase DB 연동
      </p>
      </td>
      <td style="text-align:center; font-size:14px; font-weight:bold;">
      <p align="center">
        Firebase 스크린샷
      </p>
      </td>
    </tr>
  </table>
</p>

<details>
<summary> DB table 모델 </summary>

```swift
// MARK: - Record
struct Record: Identifiable, Codable, Hashable {
    let id = UUID().uuidString
    let userLocations: [LocationCoordinate]?
    let title: String
    let currentAddress: String
    let recordingMeter: String
    let recordingTime: String

    init() {
        self.userLocations = nil
        self.title = ""
        self.currentAddress = ""
        self.recordingMeter = ""
        self.recordingTime = ""
    }

    init(userLocations: [LocationCoordinate], title: String, currentAddress: String, recordingMeter: String, recordingTime: String) {
        self.userLocations = userLocations
        self.title = title
        self.currentAddress = currentAddress
        self.recordingMeter = recordingMeter
        self.recordingTime = recordingTime
    }

    enum CodingKeys: String, CodingKey {
        case id
        case userLocations
        case title
        case currentAddress
        case recordingMeter
        case recordingTime
    }

    struct LocationCoordinate: Identifiable, Codable, Hashable {
        let id = UUID().uuidString
        let latitude: Double
        let longitude: Double

        init(coordinate: CLLocationCoordinate2D) {
            self.latitude = coordinate.latitude
            self.longitude = coordinate.longitude
        }

        var coordinate: CLLocationCoordinate2D {
            return CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
        }
    }
}
```

</details>

<details>
<summary> DB table 관리 클래스 </summary>

```swift
// MARK: - RecordStore
class RecordStore: ObservableObject {
    static let shared = RecordStore()

    init() { }

    // MARK: - 프로퍼티s
    @Published var records: [Record] = []

    let ref: DatabaseReference? = Database.database().reference()

    private let encoder = JSONEncoder()
    private let decoder = JSONDecoder()

    // MARK: - Method
    // ...

    // MARK: - startListening
    // 데이터베이스를 실시간으로 관찰하여 데이터 변경 여부를 확인
    // 실시간 데이터 read, write를 가능
    func startListening() {
        guard let dbPath = ref?.child("records") else { return }

        // MARK: Create
        dbPath.observe(DataEventType.childAdded) { [weak self] snapshot in
            guard let self = self, let json = snapshot.value as? [String: Any] else {
                return
            }
            do {
                let data = try JSONSerialization.data(withJSONObject: json)
                let record = try self.decoder.decode(Record.self, from: data)
                if !self.records.contains(where: { $0.id == record.id }) {
                    self.records.append(record)
                }
            } catch {
                print(error)
            }
        }

        // MARK: 삭제 관련
        // 데이터 삭제가 감지 되었을 때
        dbPath.observe(DataEventType.childRemoved) { [weak self] snapshot in
            guard let self = self, let json = snapshot.value as? [String: Any] else {
                return
            }

            do {
                let data = try JSONSerialization.data(withJSONObject: json)
                let record = try self.decoder.decode(Record.self, from: data)
                if let index = self.records.firstIndex(where: { $0.id == record.id }) {
                    self.records.remove(at: index)
                }
            } catch {
                print(error)
            }
        }

        // MARK: update, read 관련
        // 데이터 변경이 감지 되었을 때
        dbPath.observe(DataEventType.childChanged) { [weak self] snapshot in
            guard let self = self, let json = snapshot.value as? [String: Any] else {
                return
            }

            do {
                let data = try JSONSerialization.data(withJSONObject: json)
                let record = try self.decoder.decode(Record.self, from: data)
                if let index = self.records.firstIndex(where: { $0.id == record.id }) {
                    self.records[index] = record
                }
            } catch {
                print(error)
            }
        }
    }

    // MARK: - stopListening
    // 데이터베이스를 실시간으로 관찰하는 것을 중지
    func stopListening() {
        ref?.removeAllObservers()
    }

    // MARK: - addRecord
    // 데이터베이스에 Record 인스턴스를 추가
    func addRecord(item: Record) {
        var locationsData: [[String: Double]] = []

        if let userLocations = item.userLocations {
            for location in userLocations {
                let coordinateDict: [String: Double] = [
                    "latitude": location.latitude,
                    "longitude": location.longitude
                ]
                locationsData.append(coordinateDict)
            }
        }

        self.ref?.child("records/\(item.id)").setValue([
            "id": item.id,
            "userLocations": locationsData,
            "title": item.title,
            "currentAddress": item.currentAddress,
            "recordingMeter": item.recordingMeter,
            "recordingTime": item.recordingTime
        ])
    }
    // 데이터베이스에서 특정 경로의 데이터를 삭제
    func deleteProduct(key: String) {
        ref?.child("records/\(key)").removeValue()
    }

    // MARK: - editProduct
    // 데이터베이스에서 특정 경로의 데이터를 수정
    func editProduct(item: Record) {
        let update: [String : Any] = [
            "id": item.id,
            "userLocations": item.userLocations,
            "title": item.title,
            "currentAddress": item.currentAddress,
            "recordingMeter": item.recordingMeter,
            "recordingTime": item.recordingTime
        ]

        self.ref?.child("records/\(item.id)").setValue(update)

        if let index = self.records.firstIndex(where: { $0.id == item.id }) {
            self.records[index] = item
        }
    }
}
```

</details>

- [Firebase 관련 코드](https://github.com/BOLTB0X/MapKit-iOS/blob/main/MapKitSwiftUi02/MapKitSwiftUI/Firebase/Record.swift)

## 마커

<p align="center">
  <table style="width:100%; text-align:center; border-spacing:20px;">
    <tr>
      <td style="text-align:center; vertical-align:middle;">
        <p align="center">
        <img src="https://github.com/BOLTB0X/MapKit-iOS/blob/main/history/%EB%A7%88%EC%BB%A4%EC%BB%A4.png?raw=true" 
             alt="1" 
             style="width:200px; height:400px; object-fit:contain;"/>
        </p>
      </td>
    </tr>
    <tr>
      <td style="text-align:center; font-size:14px; font-weight:bold;">
      <p align="center">
        걸은 산책 스팟 마커로 표시
      </p>
      </td>
    </tr>
  </table>
</p>

<details>
<summary> 사용자 위치를 Mapview 에 반영 </summary>

`LocationManager` 클래스를 이용하여 현재 사용자 중심으로 맵뷰 포커스의 관리

```swift
// MARK: - LocationManager
class LocationManager: NSObject, MKMapViewDelegate, ObservableObject, CLLocationManagerDelegate {
    // MARK: 프로퍼티s
    // ...
    @Published var mapView: MKMapView = .init() // MapView
    @Published var isRecord: Bool = false // 기록중인지 확인 용도
    @Published var isChanging: Bool = false // 테스트용
    @Published var currentAddress: String = "" // 현재 주소
    @Published var recordPos: [CLLocationCoordinate2D] = [] // 기록 좌표들
    @Published var currentLatitude: CLLocationDegrees = 0.0
    @Published var currentLongitude: CLLocationDegrees = 0.0

    @Published var region = MKCoordinateRegion( // 지역
        center: .gongneungStation,
        span: MKCoordinateSpan(latitudeDelta: 0.005, longitudeDelta: 0.005)
    )

    private var manager: CLLocationManager = {
        let manager = CLLocationManager()
        manager.desiredAccuracy = kCLLocationAccuracyBest
        //manager.distanceFilter = 10
        manager.startUpdatingLocation()
        return manager
    }()

    private var currentPos: CLLocationCoordinate2D? // 현재 위치

    override init() {
        super.init()

        self.requestLocationManager()
    }

    // MARK: - Methods
    // ...

    // MARK: - requestLocationManager
    // 사용자 위치 권한 관련
    func requestLocationManager() {
        mapView.delegate = self
        manager.delegate = self

        let stauts = manager.authorizationStatus

        if stauts == .notDetermined { // 권한 요청 거질시
            manager.requestAlwaysAuthorization()
        } else if stauts == .authorizedAlways || stauts == .authorizedWhenInUse {
            mapView.showsUserLocation = true
        }
    }

    // MARK: - mapViewDidChangeVisibleRegion
    // 화면 이동될 시 이 메소드가 호출
    func mapViewDidChangeVisibleRegion(_ mapView: MKMapView) {
        DispatchQueue.main.async {
            self.isChanging = true
        }
    }

    // MARK: - mapView
    func mapView(_ mapView: MKMapView, regionDidChangeAnimated: Bool) {
        // 현재 내 맵뷰에서 중심이 되는 CLLocation
        let location: CLLocation = CLLocation(latitude: mapView.centerCoordinate.latitude, longitude: mapView.centerCoordinate.longitude)

        //self.convertLo
        DispatchQueue.main.async {
            self.isChanging = false
        }
    }

    // MARK: - mapViewFocusChange
    // 맵뷰 포커스 변경시 이용
    func mapViewFocusChange() {
        let span = MKCoordinateSpan(latitudeDelta: 0.005, longitudeDelta: 0.005)
        let region = MKCoordinateRegion(center: self.currentPos ?? .gongneungStation, span: span)

        mapView.setRegion(region, animated: true)
    }

    // MARK: - locationManagerDidChangeAuthorization
    // 사용자에게 위치 권한이 변경되면 호출
    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        if manager.authorizationStatus == .authorizedAlways || manager.authorizationStatus == .authorizedWhenInUse {
            guard let location = manager.location else {
                print("location Error")
                return
            }

            self.currentPos = location.coordinate // 현재 위치 저장
            self.mapViewFocusChange() // 저장한 위치로 이동
            self.convertLocationToAddress(location: location)
        }
    }

    // MARK: - locationManager

    // startUpdatingLocation or requestLocation 호출 했을 때 이용
    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        print("위치 변경 감지, 위치 업데이트")

        guard let location = locations.last else { return }

        currentLatitude = location.coordinate.latitude
        currentLongitude = location.coordinate.longitude

        region.center = location.coordinate
        region.span = MKCoordinateSpan(latitudeDelta: 0.005, longitudeDelta: 0.005)


        if isRecord {
            isPossibleRecord(location.coordinate)
        }

    }

    // 현재 위치 불러오는 게 실패시 호출
    func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
        print(error)
    }

    // 생략
    // ...
}
```

</details>

- [MapMarker 관련 코드](https://github.com/BOLTB0X/MapKit-iOS/tree/main/MapKitSwiftUi02/MapKitSwiftUI/MapMarker)

## 참고

- [Mapkit 공식문서](https://developer.apple.com/documentation/mapkit/mapkit_for_swiftui)

- [공식문서 - CoreLocation](https://developer.apple.com/documentation/corelocation)

- [공식문서 - CoreLocation - CLLocationManager](https://developer.apple.com/documentation/corelocation/cllocationmanager)

- [공식문서 - CoreLocation - CLGeocoder](https://developer.apple.com/documentation/corelocation/clgeocoder)

- [블로그 참조 - Working with Maps and Annotations in SwiftUI](https://www.appcoda.com/swiftui-maps/#google_vignette)

- [블로그 참조 - SwiftUI로 현재위치](https://kka3seb.tistory.com/1185)

- [블로그 참조 - MapKit](https://codekodo.tistory.com/210#1.%20%EC%82%AC%EC%9A%A9%EC%9E%90%EA%B0%80%20%EC%A7%80%EB%8F%84%EB%A5%BC%20%EC%9B%80%EC%A7%81%EC%9D%B4%EB%A9%B4%20%EC%9B%80%EC%A7%81%EC%9D%B8%20%EC%A2%8C%ED%91%9C%EC%97%90%20%EB%8C%80%ED%95%9C%20%EB%8F%84%EB%A1%9C%EB%AA%85%20%EC%A3%BC%EC%86%8C%EB%A5%BC%20%EC%8B%A4%EC%8B%9C%EA%B0%84%EC%9C%BC%EB%A1%9C%20%EA%B0%80%EC%A0%B8%EC%98%A4%EA%B8%B0-1)

- [블로그 참조 - 경로 기록하기](https://ios-developer-storage.tistory.com/entry/%EB%9F%AC%EB%8B%9D-%EC%95%B1-%EA%B0%9C%EB%B0%9C%EA%B8%B0-1-%EB%8B%AC%EB%A6%B0-%EA%B2%BD%EB%A1%9C%EB%A5%BC-%EA%B8%B0%EB%A1%9D%ED%95%98%EA%B8%B0)

- [distanceFilter](https://developer.apple.com/documentation/corelocation/cllocationmanager/1423500-distancefilter)

- [백그라운드 상태일 때 위치권한 - 블로그 참조](https://hyesunzzang.tistory.com/257)
