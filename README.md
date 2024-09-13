# PBCFoodMap
# 要記得放一個名叫icon.jpg的圖片在桌面不然打不開！
# 因為我的Google Cloud 資源有可能用完了，可能會有叫不出來的情況，需要重新申請一個新的API key

from PIL import Image, ImageTk
import tkinter as tk
from tkinter import ttk, Scale, BooleanVar, StringVar
from googlemaps import Client
import datetime
import random
from math import cos, sin, radians
import webbrowser

# Google Maps API key
gmaps = Client(key="AIzaSyAtAgKgPVGw1NTYdB75ZWhZbD-fWZlW2Sk")

# 使用者輸入的偏好
class UserPrefer():
    def __init__(self, location="", preferences=[], price=4, veg =False, opennow=False,
                 w_distance=1, w_price=1, w_food_prefer=1):
        self.location = location
        self.preferences = preferences
        self.price = price
        self.veg = veg
        self.opennow = opennow
        self.weight = [w_distance, w_price, w_food_prefer]
        self.type = "restaurant"


    def __str__(self):
        return (f"location = {self.location}\n"
                f"preferences = {self.preferences}\n"
                f"price = {self.price}\n"
                f"open_now = {self.opennow}\n"
                f"item_weight = {self.weight}")


# 首頁
class HomeWindow(tk.Frame):
    def __init__(self, master=None):
        super().__init__(master)
        self.master = master
        self.master.geometry("450x500")
        self.grid(sticky="nsew")
        self.create_widgets()


    def create_widgets(self):

        self.canvas = tk.Canvas(self, width=430, height=500, borderwidth=0)
        self.scrollbar = ttk.Scrollbar(self, orient="vertical", command=self.canvas.yview)
        self.scrollable_frame = tk.Frame(self.canvas)

        self.scrollable_frame.bind(
            "<Configure>",
            lambda e: self.canvas.configure(
                scrollregion=self.canvas.bbox("all")
            )
        )

        self.canvas.create_window((0, 0), window=self.scrollable_frame, anchor="nw")
        self.canvas.configure(yscrollcommand=self.scrollbar.set)

        self.scrollbar.pack(side=tk.RIGHT, fill="y")
        self.canvas.pack(side=tk.LEFT, fill="both", expand=True)

        
        # 設置圖標 
        self.icon_img = Image.open("C:\\Users\\USER\Desktop\\icon.jpg")  # 確保icon.png在同一目錄下，或提供正確的路徑
        self.icon_img = self.icon_img.resize((300, 300), Image.LANCZOS)
        self.icon_photo = ImageTk.PhotoImage(self.icon_img)
        self.icon_label = tk.Label(self.scrollable_frame, image=self.icon_photo)
        self.icon_label.pack(pady=10)

        # 說明文字
        info_text = (
            "歡迎使用餐廳搜尋幫手！\n\n"
            "這個程式可以幫助你根據個人偏好搜尋附近的餐廳。\n\n"
            "使用方法：\n"
            "1. 點擊下方的“開始設定”按鈕，進入偏好設定界面。\n"
            "2. 在設定界面中輸入你想要搜尋的地點、偏好類型、價位範圍等資訊。\n"
            "3. 你可以勾選“營業中”選項來只顯示當前開放的餐廳。\n"
            "4. 使用滑桿來設定不同因素的權重，如距離、價格和偏好。\n"
            "5. 點擊“提交”按鈕來查看符合條件的餐廳列表。\n"
            "6. 你也可以點擊“Spin Wheel”按鈕來隨機選擇一間餐廳。\n\n"
            "希望你能找到滿意的餐廳！"
        )
        self.info_label = tk.Label(self.scrollable_frame, text=info_text, font=("Times New Roman", 12), justify="left", wraplength=400)
        self.info_label.pack(padx=20, pady=10)

        # 啟動 InputWindow 的按鈕
        self.start_button = tk.Button(self.scrollable_frame, text="開始", command=self.open_input_window, font=("Times New Roman", 12))
        self.start_button.pack(pady=10)


    def open_input_window(self):
        input_window = tk.Toplevel(root)  # Create a new top-level window
        self.input_window = InputWindow(input_window)  # Pass the new window as the master to InputWindow


# 搜尋介面
class InputWindow():    
    def __init__(self, root):
        self.master = root
        self.master.title("請輸入您的偏好")

        """Row 1/8: Location"""
        self.location_label = tk.Label(root, text="輸入地點：")
        self.location_label.grid(row=0, column=0, padx=10, pady=10, sticky="w")
        self.entry_location = tk.Entry(root)
        self.entry_location.grid(row=0, column=1, padx=1, pady=10, sticky="w")
        self.entry_location.bind("<Return>", self.submit_info)

        """Row 2/8: Preferences"""
        self.preferences_label = tk.Label(root, text="餐廳類型：")
        self.preferences_label.grid(row=1, column=0, padx=10, pady=10, sticky="w")
        self.entry_preferences = tk.Entry(root)
        self.entry_preferences.grid(row=1, column=1, padx=1, pady=10, sticky="w")
        self.entry_preferences.bind("<Return>", self.submit_info)
        
        """Row 3/8: Special Requirements"""

        self.vegetarian_var = BooleanVar()
        self.vegetarian_check = tk.Checkbutton(root, text="素食", variable=self.vegetarian_var)
        self.vegetarian_check.grid(row=3, column=0, padx=10, pady=4, sticky="w")
        
        """Row 4/8: Open Now"""
        self.opennow_var = BooleanVar()
        self.opennow_check = tk.Checkbutton(root, text="營業中", variable=self.opennow_var)
        self.opennow_check.grid(row=3, column=1, padx=10, pady=5, sticky="w")


        """Row 5/8: Price"""
        frame_price = tk.Label(root)
        frame_price.grid(row=4, column=0, columnspan=2, padx=10, pady=2, sticky="w")

        label_price = tk.Label(frame_price, text="價位偏好：")
        label_price.grid(row=4, column=0, padx=5, pady=2, sticky="w")
        self.price_scale = Scale(frame_price, from_=1, to=4, orient=tk.HORIZONTAL)
        self.price_scale.grid(row=4, column=1, padx=3, pady=2)

        
        """Row 6/8: Rating"""
        self.rating_label = tk.Label(root, text="可接受的最低評分：")
        self.rating_label.grid(row=6, column=0, padx=10, pady=6, sticky="w")
        self.rating_var = tk.DoubleVar()
        self.rating_scale = tk.Scale(root, from_=1.0, to=5.0, resolution=0.1, orient=tk.HORIZONTAL, variable=self.rating_var)
        self.rating_scale.grid(row=6, column=1, padx=1, pady=6, sticky="w")
        
        """Row 7/8: Weight"""
        frame_weights = tk.LabelFrame(root, text="權重")
        frame_weights.grid(row=8, column=0, columnspan=2, padx=10, pady=10, sticky="we")

        label_distance_weight = tk.Label(frame_weights, text="距離權重:")
        label_distance_weight.grid(row=0, column=0, padx=5, pady=5, sticky="e")
        self.distance_weight_scale = Scale(frame_weights, from_=1, to=10, orient=tk.HORIZONTAL)
        self.distance_weight_scale.grid(row=0, column=1, padx=5, pady=5)

        label_price_weight = tk.Label(frame_weights, text="價格權重:")
        label_price_weight.grid(row=1, column=0, padx=5, pady=5, sticky="e")
        self.price_weight_scale = Scale(frame_weights, from_=1, to=10, orient=tk.HORIZONTAL)
        self.price_weight_scale.grid(row=1, column=1, padx=5, pady=5)

        label_prefer_weight = tk.Label(frame_weights, text="偏好/關鍵字權重:")
        label_prefer_weight.grid(row=2, column=0, padx=5, pady=5, sticky="e")
        self.prefer_weight_scale = Scale(frame_weights, from_=1, to=10, orient=tk.HORIZONTAL)
        self.prefer_weight_scale.grid(row=2, column=1, padx=5, pady=5)

        """Row 8/8: Submit and Spin Wheel Button"""
        self.submit_button = tk.Button(root, text="提交", command=self.submit_info)
        self.submit_button.grid(row=10, column=0, columnspan=3, padx=(10, 10), pady=(5, 5), sticky="we")
        self.submit_button.bind("<Return>", self.submit_info)

        self.spin_button = tk.Button(root, text="選擇障礙輪盤", command=self.open_spin_wheel)
        self.spin_button.grid(row=11, column=0, columnspan=3, padx=(10, 10), pady=(5, 5),sticky='we')


    def submit_info(self, event=None):
        user_location = self.entry_location.get()
        user_preferences = self.entry_preferences.get()
        user_preferences = [pre.strip() for pre in user_preferences.split(',')]
        user_price = self.price_scale.get()
        user_veg = self.vegetarian_var.get()
        user_opennow = self.opennow_var.get()
        user_distance_weight = self.distance_weight_scale.get()
        user_price_weight = self.price_weight_scale.get()
        user_prefer_weight = self.prefer_weight_scale.get()
        user_prefer = UserPrefer(location=user_location, preferences=user_preferences,
                                 veg=user_veg,
                                 price=user_price, opennow=user_opennow,
                                 w_distance=user_distance_weight,
                                 w_price=user_price_weight,
                                 w_food_prefer=user_prefer_weight)

        main_function = MainFunction()  # Create an instance of MainFunction
        restaurants = main_function.get_restaurants(user_prefer)  # Call get_restaurants
        self.display_restaurants(restaurants)  # Display the list of restaurants


    def display_restaurants(self, restaurants):
        root = tk.Tk()
        root.title("Top 20 Restaurants")

        canvas = tk.Canvas(root)
        canvas.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

        scrollbar = ttk.Scrollbar(root, orient="vertical", command=canvas.yview)
        scrollbar.pack(side=tk.RIGHT, fill="y")
        canvas.configure(yscrollcommand=scrollbar.set)

        self.restaurant_frame = tk.Frame(canvas)
        canvas.create_window((0, 0), window=self.restaurant_frame, anchor="nw")

        for idx, restaurant in enumerate(restaurants, start=1):
 
            name_button = tk.Button(self.restaurant_frame, text=restaurant['name'], font=("Times New Roman", 12, "bold"), fg="black", command=lambda url=restaurant['url']: webbrowser.open(url))
            name_button.grid(row=4*idx-4, column=1, sticky="w", padx=10, pady=5, columnspan=3)

            distance_label = tk.Label(self.restaurant_frame, text=f"與您相距: {restaurant['distance']} km")
            distance_label.grid(row=4*idx-3, column=1, sticky="w", padx=10, pady=5)

            today_weekday = datetime.datetime.now().weekday()
            today_opening_hours = restaurant['opening_hours'][today_weekday]
            opening_hours_label = tk.Label(self.restaurant_frame, text=f"今日營業時間: {today_opening_hours}")
            opening_hours_label.grid(row=4*idx-3, column=2, sticky="w", padx=10, pady=5)

            price_label = tk.Label(self.restaurant_frame, text=f"價格: {restaurant['price']}")
            price_label.grid(row=4*idx-3, column=3, sticky="w", padx=10, pady=5)

            vegetarian = '是' if '素' in restaurant['name'] or '蔬' in restaurant['name'] or 'vegan' in restaurant['name']else '可能沒有'
            vegetarian_label = tk.Label(self.restaurant_frame, text=f"提供素食餐點: {vegetarian}")
            vegetarian_label.grid(row=4*idx-2, column=1, sticky="w", padx=10, pady=5)

            separator = ttk.Separator(self.restaurant_frame, orient='horizontal')
            separator.grid(row=4*idx-1, column=0, columnspan=4, sticky="ew", pady=5)

        self.restaurant_frame.update_idletasks()
        bbox = canvas.bbox(tk.ALL)
        canvas.configure(scrollregion=bbox)


    def open_spin_wheel(self):
        spin_wheel_app = tk.Toplevel()
        SpinWheelApp(spin_wheel_app)


class MainFunction:
    def veg_check(self, name):
        for char in name:
            if char in ["素", "蔬"]:
                return True
        return False


    def get_restaurants(self, user):
        location_info = gmaps.geocode(user.location)
        latlng = location_info[0]['geometry']['location']
        veg = user.veg
        preferences = user.preferences
        opennow = user.opennow
        distance_range = [250, 750, 1000, 1500, 2000]
        place_info = []
        place_id = []

        for distance in distance_range:
            for prefers in preferences:
                places = gmaps.places_nearby(location=latlng, keyword=prefers, radius=distance, type="restaurant")
                
                for place in places["results"]:
                    # print(places['results'])
                    id = place["place_id"]
                    data = gmaps.place(place_id=id)["result"]
                    
                    try:
                        open = data["opening_hours"]["open_now"]
                    except KeyError:
                        open = False

                    if ((id not in place_id) and
                        ((open == opennow == True) or opennow == False)):
                        place_id.append(id)
                        place_info.append((distance, data))

        grading_result = []
        restaurants = []
        for info in place_info:
            distance = info[0]
            data = info[1]
            name = data["name"]
            store_veg = self.veg_check(name)
            score, details = self.grading(distance, user, data)
            grading_result.append((store_veg, score))
            restaurants.append(details)
        
        combined = list(zip(grading_result, restaurants))
        if veg == True:
            sorted_combined = sorted(combined, key=lambda x: (x[0][0], x[0][1]), reverse=True)
        else:
            sorted_combined = sorted(combined, key=lambda x: x[0][1], reverse=True)
        grading_result_sorted, restaurants_sorted = zip(*sorted_combined)
        grading_result = list(grading_result_sorted)
        restaurants = list(restaurants_sorted)
        return restaurants


    def grading(self, distance, user, data):
        weight = user.weight
        user_preferences = user.preferences
        user_price = user.price

        store_price = data.get("price_level", 2)
        url = data.get("url", "N/A")
        name = data["name"]
        types = data.get("types", "N/A")
        rating = data.get("rating", "N/A")
        opening_hours = data.get("opening_hours", "N/A")
        try:
            opening_hours = data["opening_hours"]["weekday_text"]
        except:
            opening_hours = "N/A"

        distance_score =  (2250 - distance) / 2000 * weight[0]
        price_score = (4 - (user_price - store_price)) / 4 * weight[1]
        if any(prefer in name.lower() for prefer in user_preferences):
            preference_score_temp = 1 
        else:
            preference_score_temp = 0
        preference_score = preference_score_temp * weight[2]
        rating_score = rating * sum(weight)//3
        total_score = distance_score + price_score + preference_score + rating_score

        details = {
            'name': name,
            'restaurant_type': data['types'][0] if data['types'] else 'N/A',
            'distance': round(distance / 1000, 2),
            'opening_hours': opening_hours,
            'price': store_price,
            'vegetarian': 'vegetarian' in types,
            'rating': rating,
            'score': total_score,
            'url': url
        }
 # distance_score = max(0, (2000 - distance) / 2000) * weight[0]
        # price_score = (4 - (user_price - store_price)) / 4 * weight[1]
        # preference_score = any(prefer in name.lower() for prefer in user_preferences) * weight[2]
        return total_score, details


class SpinWheelApp:
    def __init__(self, master):
        self.master = master  # 初始化Tkinter主窗口
        master.title("今天吃什麼")  # 設置窗口標題

        # 創建並放置旋轉按鈕
        self.spin_button = tk.Button(master, text="Spin", command=self.spin)
        self.spin_button.pack()

        # 創建並放置畫布，用於繪製轉盤
        self.canvas = tk.Canvas(master, width=300, height=300)
        self.canvas.pack()

        self.pointer = None  # 初始化指針變量
        self.options = ['火鍋', '拉麵', '咖喱飯', '泰式', '義大利麵', '韓式', '日式', '早午餐', '餐酒館', '小吃']  # 初始化選項列表

    def draw_wheel(self):
        self.canvas.delete("wheel")  # 繪製轉盤前先清除畫布
        angle = 36  # 每個選項固定佔36度
        start_angle = 0  # 初始化繪製起始角度

        # 繪製每個選項的弧形，並在弧形中心添加文字
        for option in self.options:
            color = "#{:06x}".format(random.randint(0, 0xFFFFFF))  # 生成隨機顏色
            self.canvas.create_arc(50, 50, 250, 250, start=start_angle, extent=angle, outline="black", width=2, style="pieslice", fill=color, tags="wheel")
            start_angle += angle  # 更新起始角度
            # 在當前弧形的中心添加文字
            text_angle = start_angle - angle / 2
            text_radius = 75  # 確保文字位置靠近中心
            x = 150 + text_radius * cos(radians(text_angle))
            y = 150 - text_radius * sin(radians(text_angle))
            self.canvas.create_text(x, y, text=option.strip(), angle=text_angle + 90, fill='white', font=("Helvetica", 12, "bold"), tags="wheel")

        if self.pointer is None:  # 如果指針不存在，則繪製指針
            self.pointer = self.canvas.create_line(150, 150, 150 + 100 * cos(radians(90)), 150 - 100 * sin(radians(90)), width=2, arrow=tk.LAST, tags="pointer")

    def spin(self):
        self.canvas.delete("result")  # 清除畫布上先前的結果
        self.draw_wheel()  # 繪製轉盤
        if not self.options:  # 如果沒有選項，則返回
            return
        
        angle_per_option = 36  # 每個選項固定36度
        spins = random.randint(1,2)  # 隨機選擇轉動的圈數，保證至少轉動1080度（3圈）
        selected_index = random.randint(0, len(self.options) - 1)  # 隨機選擇索引
        final_angle = (spins * 360) + (selected_index * angle_per_option) + random.uniform(0, angle_per_option)  # 計算最終角度

        chosen_type = self.rotate_pointer(final_angle, selected_index)  # 旋轉指針，並儲存結果
        app.input_window.entry_preferences.insert(0,
                                                  (chosen_type
                                                   if app.input_window.entry_preferences.get() == ""
                                                   else chosen_type + ","))

    def rotate_pointer(self, final_angle, selected_index):
        current_angle = 0  # 初始化當前角度
        steps = 50  # 旋轉動畫的步數
        increment = (final_angle - current_angle) / steps  # 計算每步的角度增量

        # 一步一步旋轉指針
        for step in range(steps):
            current_angle += increment  # 更新當前角度
            self.canvas.delete("pointer")  # 清除之前的指針
            # 繪製更新後角度的指針
            self.pointer = self.canvas.create_line(150, 150, 150 + 100 * cos(radians(current_angle-14.9)),
                                                   150 - 100 * sin(radians(current_angle-14.9)), width=2, arrow=tk.LAST, tags="pointer")
            self.master.update()  # 更新Tkinter窗口
            self.master.after(2)  # 暫停以實現平滑動畫
            final_angle = current_angle

        # 根據最終角度計算最終索引
        final_index = round(((final_angle + 360) % 360) / 36)
        chosen_type = self.options[final_index-1]

        return chosen_type

if __name__ == "__main__":
    root = tk.Tk()
    root.title("~歡迎來到美食獵人首頁~")
    app = HomeWindow(master=root)
    root.mainloop()
