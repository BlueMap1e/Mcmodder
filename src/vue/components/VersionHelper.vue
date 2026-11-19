<template>
  <div class="version-helper">
    <fieldset>
      <legend>从其他网站获取版本列表</legend>
      <div class="bd-callout">
        <p>
          该功能尚不保证能够准确对应版本列表与百科现有日志的版本号，对比结果仅供参考，提交日志前请仔细检查各信息是否正确~
        </p>
        <p>
          添加时请注意：新增日志的版本号格式应尽可能与现有日志统一。例如，若其他版本号有前缀“v”，则新建日志的版本号也应带此前缀~
        </p>
      </div>
      <input
        id="mcmodder-fetch-version-cf"
        ref="fetchCF"
        class="form-control"
        placeholder="输入 CFID 以查询..."
        @focus="onFetchCFFocus"
        @focusout="onFetchCFFocusout"
      />
      <input
        id="mcmodder-fetch-version-mr"
        ref="fetchMR"
        class="form-control"
        placeholder="输入 MRID 以查询..."
        @focus="onFetchMRFocus"
        @focusout="onFetchMRFocusout"
      />
      <GenericTable
        ref="table"
        :parent="parent"
        :attr="{ id: 'mcmodder-version-menu' }"
        :column-options="columnOptions"
      />
    </fieldset>
  </div>
</template>

<script setup lang="ts">
import { useTemplateRef } from "vue";
import { Mcmodder } from "../../Mcmodder";
import { TableUtils } from "../../table/Table.ts";
import { Utils } from "../../Utils.ts";
import GenericTable from "./table/GenericTable.vue";
import { GM_openInTab } from "$";

interface Props {
  parent: Mcmodder;
}

const { parent } = defineProps<Props>();

const columnOptions = {
  fileID: "文件ID",
  releaseType: [
    "发布状态",
    (data) => {
      const state = (data as string).toLowerCase();
      return `<span class="badge versiontag versiontag-${state}">${data}</span>`;
    },
  ],
  displayName: "文件名称",
  gameVersions: "支持 MC 版本",
  releaseTime: ["更新日期", TableUtils.DISPLAYRULE_DATE_MILLISEC_EN],
  mcmodVer: "对应日志版本号",
  mcmodMcver: "对应日志支持版本",
  mcmodDate: [
    "对应日志收录日期",
    (data: Date | null | undefined, row) => {
      if (!data) return "-";
      const str = data.toLocaleDateString();
      const logTime = data.valueOf();
      const releaseTime = row.releaseTime?.valueOf();
      if (!releaseTime) return "未知";
      if (Math.abs(logTime - releaseTime) <= 8.64e7) return str;
      return `<span class="mcmodder-slim-danger" data-toggle="tooltip" data-original-title="与实际更新日期存在较大误差">${str} <i class="fa fa-warning"></i></span>`;
    },
  ],
  options: [
    "操作",
    (_, data) => {
      if (data.mcmodDate || !data.displayName || !data.releaseTime) return null;
      if (data.platform === 1)
        return `<a href="/class/version/add/${Utils.abstractLastFromURL(window.location.href, "version")}/?cfid=${data.cfid}&fileid=${data.fileID}&ver=${parseCFFileName(data.displayName)}&mcver=${data.gameVersions}&date=${data.releaseTime.valueOf()}" target="_blank">补全日志</a>`;
      if (data.platform === 2)
        return `<a href="/class/version/add/${Utils.abstractLastFromURL(window.location.href, "version")}/?mrid=${data.mrid}&fileid=${data.fileID}&ver=${parseMRFileName(data.displayName)}&mcver=${data.gameVersions}&date=${data.releaseTime.valueOf()}" target="_blank">补全日志</a>`;
    },
  ],
} satisfies ColumnOptionsInitializer<GameVersionCompareEntry>;
const captchaAttemptMaxLimit = 2;
const captchaAttemptInterval = 5000;

const table = useTemplateRef("table");
const fetchCF = useTemplateRef("fetchCF");
const fetchMR = useTemplateRef("fetchMR");
let versionList = getVersionList();
let fetched = false;

function getVersionList() {
  let versionList: GameVersion[] = [];
  $(".version-content-block").each((i, e) => {
    e.id = "mcmodder-log-" + i;
    const mcRowVer = $(e).parent().attr("data-frame");
    const mcver = mcRowVer.replaceAll(" ", "").split(/[\/,&\u3001]/);
    const name = $(e).find(".name").text();
    const date = $(e).find(".time").text();
    const dateNum = date === "未知时间" ? 0 : date;
    versionList.push({
      date: new Date(dateNum),
      name: name,
      mcver: mcver,
      logid: i,
    });
  });
  return versionList;
}

// 交互为历史形态：聚焦隐式获取（每次挂载仅一次，失败无重试入口）、失焦触发查询。
// 直观性局限存在，本补丁只收口失败路径，交互重构不在范围内。
function onFetchCFFocus() {
  if (!fetched) {
    autoFillFetchID();
    fetched = true;
  }
}

function onFetchCFFocusout() {
  const cfid = fetchCF.value!.value.trim();
  if (!Number.isFinite(Number(cfid))) return;
  getCurseForgeFileList(cfid);
}

function onFetchMRFocus() {
  if (!fetched) {
    autoFillFetchID();
    fetched = true;
  }
}

function onFetchMRFocusout() {
  const mrid = fetchMR.value!.value.trim();
  if (!mrid) return;
  getModrinthFileList(mrid);
}

async function autoFillFetchID() {
  // 自动获取 CFID / MRID
  // 失败必须显式收口：请求被拒、URL 形态不符、编辑页缺字段，旧实现都会静默死亡（#128）
  try {
    const resp = await parent.utils.createRequest(
      {
        url: `${parent.hostname}/class/edit/${document.location.href.split("/version/")[1].split(".html")[0]}/`,
        method: "GET",
      },
      "从编辑页获取 CFID/MRID",
    );
    if (!resp.responseXML) {
      Utils.commonMsg("CFID/MRID 获取失败...", false);
      return;
    }
    let w = $(resp.responseXML);
    const cf = String(w.find("#class-cfprojectid").val() ?? "").trim();
    const mr = String(w.find("#class-mrprojectid").val() ?? "").trim();
    fetchCF.value!.value = cf;
    fetchMR.value!.value = mr;
    if (!cf && !mr) {
      // 404 页/无字段的页面 responseXML 仍是 truthy，静默填空串也是「没反应」的一种
      Utils.commonMsg("编辑页上没有登记 CFID/MRID，请手动填写", false);
    }
  } catch (error) {
    console.error("[Mcmodder] CFID/MRID 自动获取失败：", error);
    Utils.commonMsg("CFID/MRID 自动获取失败（编辑页不可达或未登记），请手动填写", false);
  }
}

function parseCFFileName(e: string) {
  return e
    .toLowerCase()
    .replaceAll(/forge|fabric|\.jar|alpha|beta/g, "")
    .split(/[\/-\s]/)
    .filter((k) => k)
    .slice(-1)[0];
}

function parseMRFileName(e: string) {
  return e
    .toLowerCase()
    .replaceAll(/[forge|fabric|\.jar|alpha|beta]/g, "")
    .split(/[\/-\s]/)
    .filter((k) => k)
    .slice(-1)[0];
}

/** 结束加载遮罩并提示——失败不收口会让遮罩永远停留「加载中」（#128） */
function failListLoading(message: string) {
  table.value!.completeLoading();
  Utils.commonMsg(message, false);
}

function getCurseForgeFileList(cfid: string) {
  // Number("") === 0 是有限数，空输入失焦会放行空 ID 请求
  if (!cfid) return;
  table.value!.show();
  table.value!.showLoading();
  // this.tbody.html(`<img src="${McmodderValues.assets.mcmod.loading}"></img>`);
  let fileList: CFGameVersion[] = [];
  let captchaAttempt = 0;
  let work = (index: number) => {
    parent.utils
      .createRequest(
        {
          url: `https://www.curseforge.com/api/v1/mods/${cfid}/files?pageIndex=${index}&pageSize=50&sort=dateCreated&sortDescending=true&removeAlphas=false`,
          method: "GET",
          // anonymous: true
        },
        "从 CF 获取版本列表",
      )
      .then((resp) => {
        if (resp.responseXML?.title === "Just a moment...") {
          if (captchaAttempt < captchaAttemptMaxLimit) {
            captchaAttempt++;
            Utils.commonMsg(
              `正在等待人机验证，将于 ${Utils.getFormattedTime(
                captchaAttemptInterval,
              )} 后自动重试... (${captchaAttempt}/${captchaAttemptMaxLimit})`,
            );
            setTimeout(() => {
              work(0);
            }, captchaAttemptInterval);
          } else {
            // 重试额度用尽：结束加载遮罩，交由 swal 引导手动验证
            table.value!.completeLoading();
            swal
              .fire({
                title: "验证失败",
                text: "请手动进入 CurseForge 验证页面，并于验证成功后重试。",
                showCancelButton: true,
                confirmButtonText: "前往验证",
                cancelButtonText: "取消",
              })
              .then((isConfirm) => {
                if (isConfirm.value) GM_openInTab("https://www.curseforge.com");
              });
          }
          return;
        }

        let data: { pagination?: { totalCount: number }; data?: CFGameVersion[] };
        try {
          data = JSON.parse(resp.responseText);
        } catch {
          // 拦截页不止「Just a moment...」一种标题，其余非 JSON 响应在此显式失败
          failListLoading(`CF 文件列表获取失败（HTTP ${resp.status}），响应不是有效数据`);
          return;
        }
        if (!data?.pagination || !Array.isArray(data.data)) {
          failListLoading(`CF 文件列表获取失败（HTTP ${resp.status}），响应缺少列表字段`);
          return;
        }
        if (index === 0) {
          table.value!.loadingProgress!.setMax(Math.ceil(data.pagination.totalCount / 50));
          table.value!.loadingProgress!.show();
        }
        fileList = fileList.concat(data.data);
        table.value!.loadingProgress!.setProgress(index + 1);
        if (data.pagination.totalCount > (index - 1) * 50) setTimeout(() => work(++index), 1e3);
        else {
          table.value!.empty();
          fileList.forEach((i) => {
            const releaseMap = ["-", "Release", "Beta", "Alpha"];
            let fileid = i.id;
            let releaseType = releaseMap[i.releaseType];
            let displayName = i.fileName;
            let gameVersions = i.gameVersions.join(",");
            let releaseTime = new Date(i.dateCreated);
            let mcmodVer = "未找到";
            let mcmodMcver = "-";
            let mcmodDate: Date | undefined;
            // let logid = -1;

            // 匹配百科已收录日志
            versionList.forEach((j) => {
              let mcVerName = j.name,
                prefix = mcVerName.charAt(0);
              if (["v", "V"].includes(prefix)) mcVerName = mcVerName.slice(1);
              if (parseCFFileName(displayName) === parseCFFileName(mcVerName)) {
                mcmodVer = j.name;
                mcmodMcver = j.mcver.join(",");
                mcmodDate = j.date;
                // logid = j.logid;
              }
            });
            if (mcmodDate)
              table.value!.appendData({
                platform: 1,
                cfid: cfid,
                fileID: fileid,
                releaseType: releaseType,
                releaseTime: releaseTime,
                displayName: displayName,
                gameVersions: gameVersions,
                mcmodVer: mcmodVer,
                mcmodMcver: mcmodMcver,
                mcmodDate: mcmodDate,
                options: "",
              });
          });
          table.value!.refreshAll();
        }
      })
      .catch((error) => {
        // 网络失败/超时/中止（createRequest reject）与本页处理异常统一收口
        failListLoading(error instanceof Error ? error.message : String(error));
      });
  };
  work(0);
}

function getModrinthFileList(mrid: string) {
  table.value!.show();
  table.value!.showLoading();
  // this.tbody.html(`<img src="${McmodderValues.assets.mcmod.loading}"></img>`);
  let fileList: MRGameVersion[] = [];
  let work = () => {
    parent.utils
      .createRequest(
        {
          url: `https://api.modrinth.com/v2/project/${mrid}/version`,
          method: "GET",
          anonymous: true,
        },
        "从 MR 获取版本列表",
      )
      .then((resp) => {
        let data: MRGameVersion[];
        try {
          data = JSON.parse(resp.responseText);
        } catch {
          failListLoading(`MR 文件列表获取失败（HTTP ${resp.status}），响应不是有效数据`);
          return;
        }
        if (!Array.isArray(data)) {
          failListLoading(`MR 文件列表获取失败（HTTP ${resp.status}），响应缺少列表字段`);
          return;
        }
        fileList = fileList.concat(data);
        table.value!.empty();
        fileList.forEach((i) => {
          let fileid = i.id;
          let releaseType = i.version_type;
          let displayName = i.version_number;
          let gameVersions = i.game_versions.join(",");
          let releaseTime = new Date(i.date_published);
          let mcmodVer = "未找到";
          let mcmodMcver = "-";
          let mcmodDate: Date | undefined;
          // let logid = -1;
          releaseType = releaseType.charAt(0).toUpperCase() + releaseType.slice(1);

          // 匹配百科已收录日志
          versionList.forEach((j) => {
            let mcVerName = j.name,
              prefix = mcVerName.charAt(0);
            if (["v", "V"].includes(prefix)) mcVerName = mcVerName.slice(1);
            if (parseMRFileName(displayName) === parseMRFileName(mcVerName)) {
              mcmodVer = j.name;
              mcmodMcver = j.mcver.join(",");
              mcmodDate = j.date;
              // logid = j.logid;
            }
          });
          table.value!.appendData({
            platform: 2,
            mrid: mrid,
            fileID: fileid,
            releaseType: releaseType,
            releaseTime: releaseTime,
            displayName: displayName,
            gameVersions: gameVersions,
            mcmodVer: mcmodVer,
            mcmodMcver: mcmodMcver,
            mcmodDate: mcmodDate,
            options: "",
          });
        });
        table.value!.refreshAll();
      })
      .catch((error) => {
        failListLoading(error instanceof Error ? error.message : String(error));
      });
  };
  work();
}
</script>
