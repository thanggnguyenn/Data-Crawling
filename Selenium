from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import time
import logging
import csv
import re

BASE_URL = "https://www.topcv.vn"

def parse_salary(salary_str):
    """
    Chuyển đổi chuỗi lương sang (min_salary, max_salary) đơn vị triệu VNĐ.
    Trả về (None, None) nếu là 'Thoả thuận'.
    """
    if not salary_str or any(word in salary_str.lower() for word in ['Thương lượng', 'negotiable']):
        return None, None

    # 1. Chuẩn hóa: Xóa dấu phẩy (1,000 -> 1000), đưa về chữ thường
    s = salary_str.lower().replace(',', '')
    
    # 2. Tìm tất cả các số (bao gồm cả số thập phân như 1.5)
    numbers = re.findall(r'\d+\.?\d*', s)
    numbers = [float(n) for n in numbers]
    
    if not numbers:
        return None, None

    # 3. Xác định đơn vị và tỷ lệ quy đổi (mặc định là triệu VNĐ)
    multiplier = 1.0 
    if 'usd' in s or '$' in s:
        multiplier = 0.026 # Giả sử 1 USD = 0.026 triệu VNĐ (hoặc quy đổi sang VNĐ tùy bạn)
    elif 'triệu' in s or 'tr' in s:
        multiplier = 1.0
    elif 'tỷ' in s:
        multiplier = 1000.0

    # 4. Logic tách Min - Max
    if len(numbers) >= 2:
        sal_min = numbers[0] * multiplier
        sal_max = numbers[1] * multiplier
    else:
        # Chỉ có 1 số: "Tới 30 triệu" hoặc "Trên 10 triệu"
        val = numbers[0] * multiplier
        if 'tới' in s or 'upto' in s or 'đến' in s:
            sal_min, sal_max = 0.0, val
        elif 'trên' in s or 'từ' in s or 'above' in s:
            sal_min, sal_max = val, val * 1.5 # Giả định max = 1.5 min nếu chỉ có mức sàn
        else:
            sal_min, sal_max = val, val

    return round(sal_min, 2), round(sal_max, 2)

class TopCV_Scraper:
    def __init__(self, host_name, port, filter_url: str=None, implicit_wait_time: int=15, teardown=True, log_file='topcv_scraper.log'):
        self.host_name = host_name
        self.port = port
        self.filter_url = filter_url
        self.teardown = teardown
        self.log_file = log_file
        # self.implicit_wait_time = implicit_wait_time

        # Set up the Selenium WebDriver
        self.driver = self._setup_driver()
        self.driver.implicitly_wait(implicit_wait_time)
       
        # Set up logging
        logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s', filename=log_file, filemode='w')
        self.logger = logging.getLogger()

    def _setup_driver(self):
        options = Options()
        return webdriver.Remote(command_executor=f'http://{self.host_name}:{self.port}/wd/hub', options=options)
    
    # context‑manager support
    def __enter__(self):
        # return the object that the `as …` name will bind to
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if self.teardown:
            self.driver.quit()

        if exc_type:
            print(f"An error occurred: {exc_val}")
 
    def scrape_pages(self, start_page, end_page, file_name):
        field_names = ['title', 'company', 'salary_min', 'salary_max', 'location', 'link']
        with open(file_name, 'a', encoding='utf-8') as csv_file:
            writer = csv.DictWriter(csv_file, fieldnames=field_names)
            writer.writeheader()

        try:
            for page_num in range(start_page, end_page + 1):
                url = consts.BASE_URL + self.filter_url + f"?&page={page_num}"
                self.driver.get(url)
                WebDriverWait(self.driver, 10).until(EC.presence_of_element_located((By.CSS_SELECTOR, ".job-list-search-result")))
                job_items = self.driver.find_elements(By.CSS_SELECTOR, ".job-item-search-result")
                
                for item in job_items:
                    try:
                        title = item.find_element(By.CSS_SELECTOR, "h3 > a > span").text.strip()
                        link = item.find_element(By.CSS_SELECTOR, "h3 > a").get_attribute("href")
                        salary_min, salary_max = parse_salary(item.find_element(By.CSS_SELECTOR, ".salary").text.strip())
                        job_info = {
                            'title': title,
                            'company': item.find_element(By.CSS_SELECTOR, ".company").text.strip(),
                            'salary_min': salary_min,
                            'salary_max': salary_max,
                            'location': item.find_element(By.CSS_SELECTOR, ".city-text").text.strip(),
                            'link': link
                        }
                        
                        with open(file_name, 'a', encoding='utf-8') as csv_file:
                            writer = csv.DictWriter(csv_file, fieldnames=field_names)
                            writer.writerow(job_info)
                    except Exception as e:
                        self.logger.warning(f"Failed to extract data for a job item on page {page_num} - {job_info.get('title', 'N/A')}")
                        self.logger.error(f"Error occurred: {e}")
                        continue
                
                self.logger.info(f"Hoàn thành trang {page_num}")
                time.sleep(2)
        except Exception as e:
            self.logger.error(f"Error while scraping: {e}")

if __name__ == "__main__":
    with TopCV_Scraper(host_name='localhost', port=4444, filter_url="/tim-viec-lam-cong-nghe-thong-tin-tai-ho-chi-minh-l2cr257", implicit_wait_time=20, teardown=True) as scraper:
        scraper.scrape_pages(start_page=1, end_page=25)





