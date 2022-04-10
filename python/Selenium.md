## 一、爬取B站时长

```python
from selenium import webdriver
from selenium.webdriver.support.wait import WebDriverWait
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.options import Options
from progressbar import *

bv = input("请输入BV号：")
p = int(input("请输入当前完成p："))
print("-----------------------请稍后-----------------------")

options = Options()
# 无窗口化运行
options.add_argument('--headless')
# 不输出日志
options.add_experimental_option('excludeSwitches', ['enable-logging'])
driver = webdriver.Chrome(options=options)
driver.get("https://www.bilibili.com/video/" + bv)

# 显示等待
items = WebDriverWait(driver, 30).until(lambda x: x.find_elements(By.CSS_SELECTOR, "div.clickitem"))

#'Progress: '：设置进度条前显示的文字
#Percentage() ：显示百分比
#Bar('#') ： 设置进度条形状
#ETA() ： 显示预计剩余时间
#Timer() ：显示已用时间
#FileTransferSpeed() ：显示传输速度
widgets = ['统计中: ', Percentage(), ' <<<', Bar('#'), '>>> ', Timer(), ' | ', ETA(), ' | ', FileTransferSpeed()]
bar = ProgressBar(widgets=widgets, maxval=len(items))

unwatched_hour = 0
unwatched_min = 0
unwatched_second = 0
watched_hour = 0
watched_min = 0
watched_second = 0

bar.start()
i = 0

for element in items:
    list = element.find_element(By.CSS_SELECTOR, "div.duration").text.split(":")
    if (len(list) == 3):
        unwatched_hour += int(list[0])
        unwatched_min += int(list[1])
        unwatched_second += int(list[2])
    else:
        unwatched_min += int(list[0])
        unwatched_second += int(list[1])
    if unwatched_second >= 60:
        unwatched_min += 1
        unwatched_second -= 60
    if unwatched_min >= 60:
        unwatched_hour += 1
        unwatched_min -= 60
    i += 1
    if (i == p):
        watched_hour = unwatched_hour
        watched_min = unwatched_min
        watched_second = unwatched_second
        unwatched_hour = 0
        unwatched_min = 0
        unwatched_second = 0

    bar.update(i)

bar.finish()
print("-----------------------统计完成-----------------------")
driver.quit()
print("已看 %02d:%02d:%02d 课时" % (watched_hour, watched_min, watched_second))
print("还剩 %02d:%02d:%02d 课时" % (unwatched_hour, unwatched_min, unwatched_second))

input("press enter")
```

