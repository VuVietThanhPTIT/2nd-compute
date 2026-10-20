```
root@vhhl1lab2mkolla00:~# cinder-manage version list
24.2.0




```


```
fci/dev/2024.1_2nd_gen_compute
```

```
git switch -c fci/dev/2024.1_2nd_gen_compute 24.2.0
```


```
git remote set-url origin https://gitlab.fci.vn/iaas/openstack/cinder.git


```

```

git switch -c fci/dev/2024.1_2nd_gen_compute 24.2.0
 git status
 
 
git push --set-upstream origin fci/dev/2024.1_2nd_gen_compute

 git push 
```


Đúng, công việc cốt lõi của bạn lúc này **chỉ cần tạo và viết duy nhất file `.gitlab-ci.yml`**, đặt ngay tại thư mục gốc của repository.

  

### 1. File này để làm gì?

- **Tự động hóa kiểm thử:** Thay vì mỗi lần sửa code bạn phải tự gõ lệnh test trên máy cá nhân, file này hướng dẫn GitLab tự động chạy bài kiểm tra độ phủ code (test coverage) mỗi khi có commit mới được đẩy lên.
    
      
    
- **Xác thực chất lượng mã nguồn:** Báo cáo kết quả trực tiếp lên Merge Request xem code mới có làm giảm độ phủ kiểm thử hoặc gây lỗi logic hay không.
    
      
    

### 2. File này kết nối với cái gì?

- **Kết nối với GitLab Runner:** GitLab Runner (máy chủ chạy CI nội bộ của FCI) sẽ đọc file này để biết cần kéo container nào về và thực thi các câu lệnh bên trong.
    
      
    
- **Kết nối với file `tox.ini` trong repo:** Câu lệnh `tox -e cover` trong file CI sẽ trực tiếp kích hoạt khối cấu hình `[testenv:cover]` đã được upstream OpenStack định nghĩa sẵn trong `tox.ini`.
    
      
    

### 3. File này cần những gì và nội dung ra sao?

File này cần các phần: khai báo môi trường chạy (Python), cài công cụ (`tox`), câu lệnh chạy test, và chỉ định nơi lưu file báo cáo (Artifacts).

  

Tạo file tên `.gitlab-ci.yml` ở thư mục gốc repo với nội dung:

  

YAML

```
stages:
  - test

cinder-coverage-test:
  stage: test
  image: python:3.10
  before_script:
    - python -m pip install --upgrade pip
    - pip install tox
  script:
    - tox -e cover
  coverage: '/TOTAL.+?(\d+\%)/'
  artifacts:
    name: "coverage-$CI_COMMIT_REF_NAME"
    when: always
    paths:
      - cover/
    reports:
      coverage_report:
        coverage_format: cobertura
        path: cover/coverage.xml
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^fci\/dev\/.*/'
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

### 4. Sau khi viết xong thì làm gì, như thế nào?

Bạn lưu file lại, sau đó dùng terminal đẩy file này lên nhánh GitLab:

  

1. **Thêm file vào Git tracking:**
    
      
    
    Bash
    
    ```
    git add .gitlab-ci.yml
    ```
    
2. **Tạo commit:**
    
      
    
    Bash
    
    ```
    git commit -m "ci: add gitlab ci for tox coverage test"
    ```
    
3. **Đẩy lên server GitLab:**
    
      
    
    Bash
    
    ```
    git push origin fci/dev/2024.1_2nd_gen_compute
    ```
    
4. **Kiểm tra trên giao diện web:** Mở trình duyệt vào GitLab (`gitlab.fci.vn`), vào repo Cinder, chọn mục **Build** > **Pipelines** (hoặc **CI/CD** > **Pipelines**) để xem tiến trình chạy của job.
    
      
    

### 5. Vì sao phải làm như vậy?

- GitLab chỉ kích hoạt CI khi phát hiện file cấu hình chuẩn tên `.gitlab-ci.yml` nằm trực tiếp trong nhánh được push lên server.
    
      
    
- Upstream OpenStack dùng Zuul, nhưng hệ thống nội bộ công ty dùng GitLab CI; viết file này là cách "chuyển ngữ" từ quy trình test upstream sang môi trường hạ tầng của GitLab FCI.

