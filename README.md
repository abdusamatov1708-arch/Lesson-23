# Lesson-23
import csv
import json
import re
from dataclasses import asdict, dataclass, field
from typing import Any, Dict, List
import requests


@dataclass
class Yangilik:
    id: int
    sarlavha: str
    matn: str
    sana: str
    teglar: List[str] = field(default_factory=list)
    sozlar_soni: int = 0  


def fetch_news(url: str) -> List[Dict[str, Any]]:
    """API'dan yangiliklarni yuklab oladi."""
    try:
        response = requests.get(url, timeout=10)
        response.raise_for_status() 
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"❌ API bilan bog'lanishda xatolik: {e}")
        return []


def clean_text(text: str) -> str:
    """HTML teglarni olib tashlaydi va ortiqcha bo'shliqlarni tozalaydi."""
    if not text:
        return ""
    cleaned = re.sub(r'<[^>]+>', '', text)
    cleaned = re.sub(r'\s+', ' ', cleaned).strip()
    return cleaned


def count_words(text: str) -> int:
    """Matndagi so'zlar sonini regex yordamida hisoblaydi."""
    words = re.findall(r'\b\w+\b', text)
    return len(words)


def process_news(raw_data: List[Dict[str, Any]]) -> List[Yangilik]:
    """Xom ma'lumotni qayta ishlab, Yangilik obyektlari ro'yxatiga o'tkazadi."""
    processed_list = []

    for item in raw_data:
        cleaned_matn = clean_text(item.get('matn', ''))
        soz_soni = count_words(cleaned_matn)

        yangilik = Yangilik(
            id=item.get('id'),
            sarlavha=clean_text(item.get('sarlavha', '')), 
            matn=cleaned_matn,
            sana=item.get('sana', ''),
            teglar=item.get('teglar', []),
            sozlar_soni=soz_soni
        )
        processed_list.append(yangilik)

    return processed_list


def save_to_csv(news_list: List[Yangilik], filename: str) -> None:
    """Ma'lumotlarni CSV faylga yozadi (Excel muammosiz ochishi uchun utf-8-sig)."""
    if not news_list:
        return

    fieldnames = ['id', 'sarlavha', 'matn', 'sana', 'teglar', 'sozlar_soni']

    with open(filename, mode='w', encoding='utf-8-sig', newline='') as file:
        writer = csv.DictWriter(file, fieldnames=fieldnames)
        writer.writeheader()

        for news in news_list:
            row = asdict(news)
            row['teglar'] = ", ".join(row['teglar'])
            writer.writerow(row)

    print(f"💾 Ma'lumotlar CSV ga saqlandi: {filename}")


def save_to_json(news_list: List[Yangilik], filename: str) -> None:
    """Ma'lumotlarni asdict yordamida JSON faylga yozadi."""
    json_data = [asdict(news) for news in news_list]

    with open(filename, mode='w', encoding='utf-8') as file:
        json.dump(json_data, file, ensure_ascii=False, indent=4)

    print(f"💾 Ma'lumotlar JSON ga saqlandi: {filename}")


def print_statistics(news_list: List[Yangilik]) -> None:
    """Pipeline yakunida statistika hisobotini chiqaradi."""
    if not news_list:
        print("📊 Statistika uchun ma'lumot mavjud emas.")
        return

    jami_yangilik = len(news_list)
    jami_soz = sum(news.sozlar_soni for news in news_list)

    eng_uzun_yangilik = max(news_list, key=lambda x: x.sozlar_soni)

    print("\n" + "=" * 40)
    print("📊 YANGILIKLAR PIPELINE STATISTIKASI")
    print("=" * 40)
    print(f"🔹 Jami yangiliklar soni: {jami_yangilik} ta")
    print(f"🔹 Jami so'zlar soni    : {jami_soz} ta")
    print(
        f"🔹 Eng uzun yangilik   : ID: {eng_uzun_yangilik.id} (Sarlavha: '{eng_uzun_yangilik.sarlavha[:30]}...', {eng_uzun_yangilik.sozlar_soni} ta so'z)")
    print("=" * 40)


if __name__ == "__main__":
    API_URL = "https://jsonplaceholder.typicode.com/posts"  

    mock_api_data = [
        {
            "id": 1,
            "sarlavha": "<h1>Tezkor: Python 3.12 Chiqdi!</h1>",
            "matn": "<p>Bugun Python hamjamiyati   yangi versiyani taqdim etdi. Unda ko'plab <br> yangiliklar bor.</p>",
            "sana": "2026-06-19",
            "teglar": ["python", "texnologiya"]
        },
        {
            "id": 2,
            "sarlavha": "Ob-havo ma'lumoti",
            "matn": "Ertaga   yomg'ir yog'ishi kutilmoqda. Soyabon olishni unutmang.  Havo harorati 25 daraja bo'ladi.",
            "sana": "2026-06-20",
            "teglar": ["obhavo", "jamiyat"]
        }
    ]

    print("🚀 Pipeline ishga tushdi...")

    raw_data = mock_api_data

    processed_news = process_news(raw_data)

    save_to_csv(processed_news, "yangiliklar.csv")
    save_to_json(processed_news, "yangiliklar.json")

    print_statistics(processed_news)
