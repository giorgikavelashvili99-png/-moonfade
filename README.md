# მთვარისფერი — WeTransfer-ის მსგავსი ვიდეო-გაზიარების სერვისი

ფაილი ინახება Cloudflare R2-ში **ურედაქტირებლად** (არანაირი კომპრესია/ტრანსკოდირება — ხარისხი უცვლელია).
ბრაუზერი პირდაპირ R2-ში ტვირთავს/წერს presigned URL-ების საშუალებით — Render-ის სერვერი ფაილის ბაიტებს საერთოდ არ ეხება, მხოლოდ ბმულებს აგენერირებს.

## 1. Cloudflare R2 (უფასო storage)

1. cloudflare.com → R2 → შექმენი bucket, მაგ. `moonfade`.
2. **R2 → Manage API tokens** → შექმენი token წვდომით ამ bucket-ზე (Object Read & Write). ჩაინიშნე:
   - Account ID
   - Access Key ID
   - Secret Access Key
3. **CORS policy** bucket-ზე (აუცილებელია, რომ ბრაუზერმა პირდაპირ შეძლოს ატვირთვა):

```json
[
  {
    "AllowedOrigins": ["https://your-app.onrender.com"],
    "AllowedMethods": ["PUT", "GET"],
    "AllowedHeaders": ["*"],
    "MaxAgeSeconds": 3000
  }
]
```

4. **Lifecycle rule (backstop)** — Settings → Object Lifecycle Rules → "Delete objects after 1 day". ეს ზედმეტი დაცვაა იმ შემთხვევისთვის, თუ აპლიკაცია დროებით ჩავარდა და cleanup thread-მა ვერ იმუშავა; ძირითადი 6-საათიანი წაშლა ხდება `app.py`-ის background thread-ით.

## 2. Resend (უფასო ტრანზაქციული ემეილი — 3000/თვეში, 100/დღეში)

1. resend.com → დარეგისტრირდი → API key შექმენი.
2. სატესტოდ გამოსადეგია ნაგულისხმევი გამგზავნი `onboarding@resend.dev` — საკუთარი დომენის დასადასტურებლად შემდეგში დაამატებ.

## 2.5 ლოკალური ტესტირება

`.env` ფაილში უკვე ჩაწერილია თქვენი R2 access key/secret — ეს ფაილი **მხოლოდ თქვენს კომპიუტერზეა**, `.gitignore`-ში უკვე გამორთულია, ანუ `git push`-ის დროს GitHub-ზე ატვირთვის საშიშროება არ არსებობს.

გაშვება ლოკალურად:
```
pip install -r requirements.txt
python app.py
```
გახსენით `http://localhost:5000`.

**უსაფრთხოების შენიშვნა:** Access key/secret-ი არასდროს უნდა ჩაწეროთ პირდაპირ `app.py`-ში ან README-ში, თუ ეს რეპო საჯაროდ აიტვირთება GitHub-ზე — ყოველთვის `.env`-ის ან Render-ის dashboard-ის environment variables-ის საშუალებით გადაეცით. თუ ოდესმე საეჭვოდ მოგეჩვენებათ, რომ key სადმე გამჟღავნდა, Cloudflare-ის R2 dashboard-იდან ნებისმიერ დროს შეგიძლიათ ძველი token გააუქმოთ და ახალი გამოქმნათ.

## 3. Render.com deploy

1. ატვირთე ეს რეპო GitHub-ზე.
2. Render → New → Web Service → აირჩიე რეპო.
3. Build command: `pip install -r requirements.txt`
   Start command: `gunicorn app:app`
4. Environment variables:

| ცვლადი | მნიშვნელობა |
|---|---|
| `R2_ACCOUNT_ID` | უკვე ჩაშენებულია კოდში (`556af970dee894bf159fb78830fb44e8`) — env-ით შეგიძლია გადააწერო, თუ სხვა ანგარიშზე გადახვალ |
| `R2_ACCESS_KEY_ID` | Cloudflare-დან — **აუცილებელია**, ეს secret-ია და კოდში არ ინახება |
| `R2_SECRET_ACCESS_KEY` | Cloudflare-დან — **აუცილებელია**, ეს secret-ია და კოდში არ ინახება |
| `R2_BUCKET` | უკვე ჩაშენებულია (`moonfade`) |
| `BASE_URL` | `https://your-app.onrender.com` |
| `EXPIRY_HOURS` | `6` |
| `RESEND_API_KEY` | Resend-დან |
| `RESEND_FROM` | `onboarding@resend.dev` |
| `MAX_UPLOAD_BYTES` | `4294967296` (4GB, სურვილისამებრ) |

**შენიშვნა:** Render-ის free tier-ზე disk არ ინახება perm-ად deploy-ებს შორის — ეს პრობლემა არ არის, რადგან `moonfade.db` (SQLite) მხოლოდ მეტამონაცემებს (ფაილის სახელი, ვადა) ინახავს, არა თავად ვიდეოებს. თუ გინდა, restart-ებზეც გადარჩეს ისტორია, მომავალში Render-ის free Postgres-ზე გადართვა შეიძლება.

## როგორ რჩება 10GB-ის ფარგლებში

6-საათიანი ვადა ნიშნავს, რომ ერთდროულად "ცოცხალი" ფაილების საერთო მოცულობა თითქმის არასდროს გადააჭარბებს რეალურ დღიურ ტრაფიკს — მხოლოდ ბოლო 6 საათის ატვირთვები ინახება ერთდროულად, დანარჩენი ავტომატურად იშლება.

## შემდეგი ნაბიჯები (სურვილისამებრ)

- ატვირთვამდე ფაილის ზომის/ტიპის დამატებითი ვალიდაცია frontend-ზეც
- multi-file ატვირთვა ერთ ბმულში (ZIP-ის მაგივრად, prefix-ით)
- rate limiting ბოროტად გამოყენების თავიდან ასაცილებლად
