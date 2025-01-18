# ToolsKit

```python
import json
import os
import re
from pathlib import Path
import requests
from collections import defaultdict
from concurrent.futures import ThreadPoolExecutor
from urllib.parse import urlparse
import time

def merge_json_files(root_dir):
    """合并所有JSON文件"""
    all_data = defaultdict(list)

    for dirpath, dirnames, filenames in os.walk(root_dir):
        current_dir = os.path.basename(dirpath)

        for filename in filenames:
            if filename.endswith('.json'):
                file_path = os.path.join(dirpath, filename)
                try:
                    with open(file_path, 'r', encoding='utf-8') as f:
                        data = json.load(f)
                        data['source_directory'] = current_dir
                        data['source_file'] = filename
                        all_data[data['date']].append(data)
                except json.JSONDecodeError:
                    print(f"Error reading file: {file_path}")
                except Exception as e:
                    print(f"Error processing file {file_path}: {str(e)}")

    final_data = {
        "all_records": []
    }

    for date in sorted(all_data.keys()):
        final_data["all_records"].extend(all_data[date])

    final_data["metadata"] = {
        "total_records": len(final_data["all_records"]),
        "date_range": {
            "start": min(all_data.keys()),
            "end": max(all_data.keys())
        },
        "source_directories": list(set(record['source_directory']
                                     for record in final_data["all_records"]))
    }

    return final_data

def extract_alicdn_links(json_data):
    """提取JSON数据中的alicdn链接"""
    links = []
    for record in json_data.get('all_records', []):
        for chat in record.get('chats', []):
            content = chat.get('content', '')
            if 'https://img.alicdn.com' in content:
                matches = re.findall(r'https://img\.alicdn\.com[^\s\'"]+', content)
                for match in matches:
                    links.append({
                        'url': match,
                        'date': record.get('date', ''),
                        'time': chat.get('time', ''),
                        'name': chat.get('name', '')
                    })
    return links

def download_image(link_info):
    """下载单个图片"""
    url = link_info['url']
    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
        'Referer': 'https://www.taobao.com',
        'Accept': 'image/webp,image/apng,image/*,*/*;q=0.8',
        'Accept-Encoding': 'gzip, deflate, br',
        'Accept-Language': 'zh-CN,zh;q=0.9,en;q=0.8',
        'Cache-Control': 'no-cache',
    }

    try:
        date_dir = Path('downloads') / link_info['date']
        date_dir.mkdir(parents=True, exist_ok=True)

        filename = os.path.basename(urlparse(url).path)
        if not filename.lower().endswith(('.jpg', '.jpeg', '.png', '.gif')):
            filename += '.jpg'

        save_path = date_dir / filename

        if save_path.exists():
            print(f"文件已存在，跳过: {save_path}")
            return True

        # 添加重试机制
        max_retries = 3
        for attempt in range(max_retries):
            try:
                response = requests.get(url, headers=headers, timeout=10)
                response.raise_for_status()

                # 检查响应内容类型
                if 'image' not in response.headers.get('Content-Type', ''):
                    print(f"非图片响应: {url}")
                    return False

                with open(save_path, 'wb') as f:
                    f.write(response.content)

                print(f"成功下载: {save_path}")
                return True

            except requests.RequestException as e:
                if attempt < max_retries - 1:
                    time.sleep(2 ** attempt)  # 指数退避
                    continue
                print(f"下载失败 {url}: {str(e)}")
                return False

    except Exception as e:
        print(f"处理失败 {url}: {str(e)}")
        return False

def main():
    # 1. 合并JSON文件
    print("开始合并JSON文件...")
    root_dir = "."  # 可以修改为实际的根目录路径
    merged_data = merge_json_files(root_dir)

    # 保存合并后的JSON文件
    merged_file = "merged_chat_records.json"
    with open(merged_file, 'w', encoding='utf-8') as f:
        json.dump(merged_data, f, ensure_ascii=False, indent=2)

    print("\nJSON合并完成！统计信息：")
    print(f"总记录数: {merged_data['metadata']['total_records']}")
    print(f"日期范围: {merged_data['metadata']['date_range']['start']} 到 "
          f"{merged_data['metadata']['date_range']['end']}")
    print("来源目录:")
    for dir_name in merged_data['metadata']['source_directories']:
        print(f"  - {dir_name}")

    # 2. 提取并下载图片
    print("\n开始提取图片链接...")
    links = extract_alicdn_links(merged_data)

    # 保存链接到文件
    links_file = "alicdn_links.json"
    with open(links_file, 'w', encoding='utf-8') as f:
        json.dump(links, f, ensure_ascii=False, indent=2)

    print(f"找到 {len(links)} 个图片链接，已保存到 {links_file}")


    # 修改下载部分
    print("\n开始下载图片...")
    Path('downloads').mkdir(exist_ok=True)

    # 减少并发数，添加延迟
    with ThreadPoolExecutor(max_workers=3) as executor:
        futures = []
        for link in links:
            # 添加随机延迟，避免请求过于密集
            time.sleep(0.5)
            futures.append(executor.submit(download_image, link))

        # 收集结果
        results = []
        for future in futures:
            try:
                results.append(future.result())
            except Exception as e:
                print(f"任务执行失败: {str(e)}")
                results.append(False)

    # 打印下载统计信息
    success_count = sum(1 for r in results if r)
    print(f"\n下载完成！")
    print(f"成功: {success_count}")
    print(f"失败: {len(links) - success_count}")

    # 保存失败的链接
    failed_links = [link for link, result in zip(links, results) if not result]
    if failed_links:
        with open('failed_downloads.json', 'w', encoding='utf-8') as f:
            json.dump(failed_links, f, ensure_ascii=False, indent=2)
        print(f"\n失败的下载已保存到 failed_downloads.json")

if __name__ == "__main__":
    # 确保已安装必要的库：
    # pip install requests
    main()
```

```shell
# 设置目标文件夹路径
TARGET_FOLDER="merged_images"

# 创建目标文件夹
mkdir -p "$TARGET_FOLDER"

# 遍历所有子文件夹并移动图片
find . -type f \( -iname "*.jpg" -o -iname "*.jpeg" -o -iname "*.png" -o -iname "*.gif" -o -iname "*.bmp" \) -exec mv {} "$TARGET_FOLDER" \;

echo "所有图片已移动到 $TARGET_FOLDER 文件夹中"
```
