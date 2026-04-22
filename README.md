# zenedu-pipedrive
function doPost(e) {
  try {
    if (!e || !e.postData || !e.postData.contents) {
      return ContentService.createTextOutput("No post data");
    }

    const payload = JSON.parse(e.postData.contents);
    Logger.log(JSON.stringify(payload, null, 2));

    const status = payload?.data?.status ?? payload?.status ?? "";
    const email = payload?.data?.email ?? payload?.email ?? "";

    const amount =
      payload?.data?.amount ??
      payload?.data?.price ??
      payload?.data?.plan?.price ??
      payload?.data?.subscription?.price ??
      payload?.data?.offer?.price ??
      payload?.data?.order?.amount ??
      payload?.amount ??
      payload?.price ??
      payload?.order?.amount ??
      "-";

    const currency =
      payload?.data?.currency ??
      payload?.data?.plan?.currency ??
      payload?.data?.subscription?.currency ??
      payload?.data?.offer?.currency ??
      payload?.data?.order?.currency ??
      payload?.currency ??
      payload?.order?.currency ??
      "";

    if (!["paid", "failed", "cancelled"].includes(status)) {
      return ContentService.createTextOutput("Ignored");
    }

    if (!email) {
      return ContentService.createTextOutput("No email");
    }

    const apiToken = "";
    const clubPipelineId = 4;
    const baseUrl = "https://api.pipedrive.com/v1";

    const personSearchUrl =
      `${baseUrl}/persons/search?term=${encodeURIComponent(email)}` +
      `&fields=email&exact_match=true&api_token=${apiToken}`;

    const personSearchResp = UrlFetchApp.fetch(personSearchUrl, {
      method: "get",
      muteHttpExceptions: true,
    });

    const personSearchCode = personSearchResp.getResponseCode();
    const personSearchText = personSearchResp.getContentText();
    Logger.log(`Person search response (${personSearchCode}): ${personSearchText}`);

    if (personSearchCode < 200 || personSearchCode >= 300) {
      return ContentService.createTextOutput(`Person search failed: ${personSearchCode}`);
    }

    const personSearchData = JSON.parse(personSearchText);

    if (!personSearchData.data?.items?.length) {
      return ContentService.createTextOutput("No person found");
    }

    const personId = personSearchData.data.items[0].item.id;

    const dealsUrl = `${baseUrl}/persons/${personId}/deals?api_token=${apiToken}`;
    const dealsResp = UrlFetchApp.fetch(dealsUrl, {
      method: "get",
      muteHttpExceptions: true,
    });

    const dealsCode = dealsResp.getResponseCode();
    const dealsText = dealsResp.getContentText();
    Logger.log(`Deals response (${dealsCode}): ${dealsText}`);

    if (dealsCode < 200 || dealsCode >= 300) {
      return ContentService.createTextOutput(`Deals fetch failed: ${dealsCode}`);
    }

    const dealsData = JSON.parse(dealsText);

    if (!dealsData.data?.length) {
      return ContentService.createTextOutput("No deals found");
    }

    const clubDeals = dealsData.data.filter(d => Number(d.pipeline_id) === Number(clubPipelineId));

    if (!clubDeals.length) {
      return ContentService.createTextOutput("No deals in Club pipeline");
    }

    const deal =
      clubDeals.find(d => d.status === "open") ||
      clubDeals.sort((a, b) => new Date(b.add_time) - new Date(a.add_time))[0];

    if (!deal || !deal.id) {
      return ContentService.createTextOutput("No valid deal found");
    }

    const dealId = deal.id;
    const amountText = currency ? `${amount} ${currency}` : `${amount}`;

    let noteText = "";

    if (status === "paid") {
      noteText = `✅ Оплата получена<br>Email: ${email}<br>Сумма: ${amountText}`;
    } else if (status === "failed") {
      noteText = `❌ Оплата не прошла<br>Email: ${email}<br>Сумма: ${amountText}`;
    } else if (status === "cancelled") {
      noteText = `⚠️ Оплата отменена<br>Email: ${email}<br>Сумма: ${amountText}`;
    }

    const noteUrl = `${baseUrl}/notes?api_token=${apiToken}`;
    const noteResp = UrlFetchApp.fetch(noteUrl, {
      method: "post",
      contentType: "application/json",
      muteHttpExceptions: true,
      payload: JSON.stringify({
        content: noteText,
        deal_id: dealId,
      }),
    });

    const noteCode = noteResp.getResponseCode();
    const noteTextResp = noteResp.getContentText();
    Logger.log(`Note response (${noteCode}): ${noteTextResp}`);

    if (noteCode < 200 || noteCode >= 300) {
      return ContentService.createTextOutput(`Note create failed: ${noteCode}`);
    }

    return ContentService.createTextOutput("OK");
  } catch (err) {
    Logger.log(`Error: ${err.message}\n${err.stack}`);
    return ContentService.createTextOutput(`Error: ${err.message}`);
  }
}
