# Gmailの予約メールをGoogleカレンダーに自動登録し通知するGAS
## ( ..)φメモメモ

Google Apps Script (GAS) を使って、Gmailに届いた特定の予約メールを自動的にGoogleカレンダーに登録し、自分に通知メールを送信するGASを作成しました。本記事では、その仕組みと実装方法を紹介します。

---

## 背景

日々のスケジュール管理を効率化するために、予約メールを手動でカレンダーに登録する手間を省きたいと考えました。そこで、Gmailに届いた予約完了メールをトリガーにして、Googleカレンダーに予定を自動登録し、さらに通知メールを送信する仕組みを構築しました。

---

## 実装の概要

以下の機能を実現するGASを作成しました：

1. Gmailから特定の条件に一致するメールを検索。
2. メール本文から予約日時を抽出。
3. Googleカレンダーに予定を登録。
4. 登録完了後、自分に通知メールを送信。
5. 処理済みのメールをアーカイブ。

---

## コード

以下が実際のコードです。個人情報（メールアドレスやURL）は伏せています。

```javascript
// メールが来たらGoogleカレンダーに追加し、自分に通知メールを送信する
function addSalonAppointmentToCalendar() {
  const threads = GmailApp.search('from:****@****.jp subject:"予約登録が完了しました" newer_than:7d');
  const calendar = CalendarApp.getDefaultCalendar();
  const userEmail = Session.getActiveUser().getEmail();

  threads.forEach(thread => {
    const messages = thread.getMessages();
    messages.forEach(message => {
      const body = message.getPlainBody();

      const datetimeMatch = body.match(/ご予約日時:\s*(\d{4})年(\d{2})月(\d{2})日\s*(\d{2})時(\d{2})分/);
      if (!datetimeMatch) return;

      const [_, year, month, day, hour, minute] = datetimeMatch;
      let startTime = new Date(Number(year), Number(month) - 1, Number(day), Number(hour), Number(minute));
      const endTime = new Date(startTime.getTime() + 60 * 60 * 1000); // 1時間後

      // 重複チェック
      const events = calendar.getEvents(startTime, endTime);
      const alreadyExists = events.some(event => event.getTitle() === "サロン予約");
      if (alreadyExists) {
        Logger.log("すでにイベントがあるため中断");
        return;
      }

      const mapUrl = generateMapUrlWithArrivalTime(startTime); // 10分前到着の地図URL
      Logger.log("mapUrl: ");
      Logger.log(mapUrl);

      // イベント追加
      calendar.createEvent("サロン予約", startTime, endTime, {
        location: mapUrl
      });

      // 通知メール送信
      GmailApp.sendEmail(
        userEmail,
        "【通知】カレンダーに予約イベントを追加しました",
        `以下の内容で予定を追加しました。\n\nタイトル: サロン予約\n日時: ${startTime.toLocaleString("ja-JP")}\n地図URL: ${mapUrl}`
      );
    });

    // スレッドをアーカイブ
    thread.moveToArchive();
  });

  Logger.log("完了：メールが来たらGoogleカレンダーに追加＆通知メール送信");
}

// 10分前到着の地図URLを生成（JST補正含む）
function generateMapUrlWithArrivalTime(date) {
  const unixTime = Math.floor(date.getTime() / 1000) - 10 * 60 + 9 * 60 * 60;
  return `https://www.google.co.jp/maps/dir/......経路の始点と終点を表す部分......!8j${unixTime}!3e3?hl=ja&entry=ttu`;
}
```
