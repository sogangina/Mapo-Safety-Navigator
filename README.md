# Mapo-Safety-Navigator
2024 Sogang ✕ Upstage University LLM Project Course
import pandas as pd
import requests
import json
from geopy.distance import geodesic

# API 키와 URL 설정
api_key = "QLdNjlGyula0zcHRE3c6T3SN14Q3ewnn6axyDT5M"
geo_url = "https://apis.openapi.sk.com/tmap/geo/fullAddrGeo"
route_url = "https://apis.openapi.sk.com/tmap/routes/pedestrian?version=1&format=json"
distance_url = "https://apis.openapi.sk.com/tmap/routes/distance"

headers = {
    "appKey": api_key,
    "Content-Type": "application/json"
}

# 출발지, 도착지 주소
start = "서울 마포구 백범로 35 서강대학교"
end = "서울 마포구 와우산로 94"

# 출발지 좌표 요청
start_params = {
    "version": 1,
    "fullAddr": start,
    "coordType": "WGS84GEO",
    "appKey": api_key
}
start_response = requests.get(geo_url, params=start_params)
start_data = start_response.json()
startX = start_data["coordinateInfo"]["coordinate"][0]["newLon"] #경도
startY = start_data["coordinateInfo"]["coordinate"][0]["newLat"] #위도 

# 도착지 좌표 요청
end_params = {
    "version": 1,
    "fullAddr": end,
    "coordType": "WGS84GEO",
    "appKey": api_key
}
end_response = requests.get(geo_url, params=end_params)
end_data = end_response.json()
endX = end_data["coordinateInfo"]["coordinate"][0]["newLon"] # 경도
endY = end_data["coordinateInfo"]["coordinate"][0]["newLat"] # 위도 

print(f"출발지 좌표: {startX}, {startY}")
print(f"도착지 좌표: {endX}, {endY}")

# 경로 탐색 요청 파라미터
route_params = {
    "startX": startX,
    "startY": startY,
    "endX": endX,
    "endY": endY,
    "reqCoordType": "WGS84GEO",
    "resCoordType": "WGS84GEO",  # WGS84GEO 좌표계로 변경
    "startName": "출발지",
    "endName": "도착지",
}
# 경로 탐색 API 호출
response = requests.post(route_url, headers=headers, json=route_params)
if response.status_code == 200:
    data = response.json()
    
    # 총 거리, 시간
    total_distance = data["features"][0]["properties"]["totalDistance"]
    total_time_sec = data["features"][0]["properties"]["totalTime"]
    total_time_min = total_time_sec // 60 # 분
    total_time_sec = total_time_sec % 60 # 초
    print(f"총 거리: {total_distance} 미터")
    print(f"총 예상 시간: {total_time_min}분 {total_time_sec}초")

    # 각 구간별 세부 정보 (설명, 거리, 시간, 위도/경도)
    for feature in data["features"]:
        properties = feature["properties"]
        geometry = feature["geometry"]
        
        # 구간 설명, 거리 및 시간
        description = properties.get("description", "N/A")
        distance = properties.get("distance", None)
        time = properties.get("time", None)
        
        # 거리와 시간이 없을 경우 조건 처리
        distance_text = f"{distance}미터" if distance is not None else "정보 없음"
        
        # 시간 형식 변경: 60초 이상일 경우 분과 초로 변환
        if time is not None:
            if time >= 60:
                minutes = time // 60
                seconds = time % 60
                time_text = f"{minutes}분 {seconds}초"
            else:
                time_text = f"{time}초"
        else:
            time_text = "정보 없음"
        
        # 구간 위도와 경도 (모든 좌표 포함)
        coordinates = []
        if geometry["type"] == "Point":
            coordinates.append(f"({geometry['coordinates'][1]}, {geometry['coordinates'][0]})")
        elif geometry["type"] == "LineString":
            for coord in geometry["coordinates"]:
                coordinates.append(f"({coord[1]}, {coord[0]})")
        
        # 출력 형식: 구간 설명, 거리, 시간, 모든 좌표
        coordinates_text = ", ".join(coordinates)
        print(f"구간 설명: {description}, 거리: {distance_text}, 예상 시간: {time_text}, 위치: {coordinates_text}")
else:
    print("API 요청 실패:", response.status_code)

# 출발지와 도착지 위도, 경도 저장
coordinates = {
    "출발지": (startY, startX),
    "도착지": (endY, endX),
    "구간": []
}

# JSON 결과에서 각 구간의 위도와 경도 추출
for feature in data["features"]:
    geometry = feature["geometry"]

    # Point 타입일 경우 해당 좌표를 추가
    if geometry["type"] == "Point":
        lat, lon = geometry["coordinates"][1], geometry["coordinates"][0]
        coordinates["구간"].append((lat, lon))
    
    # LineString 타입일 경우 모든 좌표를 추가
    elif geometry["type"] == "LineString":
        for coord in geometry["coordinates"]:
            lat, lon = coord[1], coord[0]
            coordinates["구간"].append((lat, lon))

            
latlon_list = []
# 최종 위도와 경도만 출력
print("출발지:", coordinates["출발지"])
print("도착지:", coordinates["도착지"])
print("구간 좌표:")

for latlon in coordinates["구간"]:
    latlon_list.append(latlon)
print(latlon_list)


# CCTV 데이터 로드
cctv_data = pd.read_csv(r"C:\Users\82102\OneDrive\바탕 화면\마포구_길찾기\서울시 마포구 안심이 CCTV 연계 현황.csv", encoding="cp949")

# CCTV 좌표 추출 (위도, 경도 형식의 리스트로 저장)
cctv_coords = list(zip(cctv_data['위도'], cctv_data['경도']))

# 각 CCTV에서 가장 가까운 경로 좌표와의 거리 계산 함수
def find_closest_points(cctv_coords, latlon_list, n):
    closest_points = []
    seen_cctvs = set()  # 중복된 CCTV 좌표를 확인하기 위한 집합

    for cctv in cctv_coords:
        distances = []
        
        # 각 경로 좌표와의 거리 계산
        for route_point in latlon_list:
            distance = geodesic(cctv, route_point).meters  # geopy로 거리 계산
            distances.append((distance, route_point))
        
        # 각 CCTV와 가장 가까운 경로 좌표를 거리 기준으로 정렬
        distances.sort(key=lambda x: x[0])
        
        # 중복되지 않으면서 거리 15m 이상인 경우만 추가
        if cctv not in seen_cctvs and distances[0][0] >= 15:
            closest_points.append((distances[0][0], cctv, distances[0][1]))  # (최단 거리, CCTV 좌표, 가장 가까운 경로 좌표)
            seen_cctvs.add(cctv)  # 추가한 CCTV 좌표를 중복 확인 집합에 추가

    # 전체 CCTV에 대해 가장 가까운 거리 상위 n개 선택
    closest_points.sort(key=lambda x: x[0])  # 거리 기준으로 정렬
    return closest_points[:n]

# 가까운 CCTV n개 찾기
n = int(input("가장 가까운 CCTV 몇 개를 찾으시겠습니까? "))
closest_results = find_closest_points(cctv_coords, latlon_list, n)

# 결과 출력
print(f"\nCCTV 좌표별로 가장 가까운 거리 상위 {n}개:")
for i, (distance, cctv, route_point) in enumerate(closest_results, 1):
    print(f"{i}. 거리: {distance:.2f}미터")
    print(f"   CCTV 위치: {cctv}")
    print(f"   가장 가까운 경로 좌표: {route_point}")
    print("-" * 20)
    
