# note

ae8129d94661d47dd2fb843b52ee258e

/**
 * Created by ouyangjingzhou@jianyi.tech on 2017/1/1.
 */
import {Injectable} from "@angular/core";
import {Headers, Http, RequestMethod, RequestOptionsArgs, Response, ResponseContentType} from "@angular/http";
import {ToolsService} from "./tools.service";
import {DisabledRest} from "../meta/disabled.meta";
import {DisabledService} from "./disabled.service";
import {ErrorCodeMapType, ErrorCodeType, HttpOptionsArgs} from "../meta/http.meta";
import {AlertService} from "./alert.service";
import {isDate, isFunction, isString} from "util";
import {HttpProvider} from "../meta/provider.meta";
import {Icon} from "../meta/common.meta";

@Injectable()
export class HttpService {

  constructor(private http: Http,
              private toolsService: ToolsService,
              private disabledService: DisabledService,
              private alertService: AlertService) {
  }

  private baseUrl: string;
  private httpProvider: HttpProvider;
  private preventRepetitionParamsRecord: { [index: string]: string } = {};
  private preventRepetitionRandomId: { [index: string]: string } = {};

  setProvider(httpProvider: HttpProvider) {
    if (!httpProvider) {
      return;
    }
    this.httpProvider = httpProvider;
    // 获取服务器地址 处理拼接时的斜杠
    this.baseUrl = httpProvider.getBaseUrl();
    if (this.baseUrl.endsWith("/")) {
      this.baseUrl = this.baseUrl.substring(0, this.baseUrl.length - 1);
    }
  }

  get(options: HttpOptionsArgs): void {
    options.method = RequestMethod.Get;
    this.request(options);
  }

  post(options: HttpOptionsArgs): void {
    options.method = RequestMethod.Post;
    this.request(options);
  }

  del(options: HttpOptionsArgs): void {
    options.method = RequestMethod.Delete;
    this.request(options);
  }

  put(options: HttpOptionsArgs): void {
    options.method = RequestMethod.Put;
    this.request(options);
  }

  download(url: string, data: any = {}) {
    data["user-token"] = this.httpProvider.getDownloadUserToken();
    window.location.href = this.toolsService.getFullUrl(this.baseUrl, url) + '?' + this.urlParamsTranslate(data);
  }

  request(options: HttpOptionsArgs) {
    if (this.httpProvider.requestIntercept(options)) {
      return;
    }

    if (options.preventRepetition || options.preventRepetitionId) {
      options.preventRepetitionId = options.preventRepetitionId ? options.preventRepetitionId : options.url;
      let lastParamsJson: string = this.preventRepetitionParamsRecord[options.preventRepetitionId];
      let paramsJson: string = "";
      if (options.body) {
        paramsJson += JSON.stringify(options.body);
      }
      if (options.params) {
        paramsJson += JSON.stringify(options.params);
      }
      if (lastParamsJson && lastParamsJson === paramsJson) {
        return;
      }
      this.preventRepetitionParamsRecord[options.preventRepetitionId] = paramsJson;
      options.preventRepetitionRandomId = this.toolsService.getRandomId();
      this.preventRepetitionRandomId[options.preventRepetitionId] = options.preventRepetitionRandomId;
    }

    // 当路由跳转后 之前如果存在有还没回调的请求则忽略所有的回调 不管成功还是失败
    this.setDefaultOptions(options);
    let disabledRest: DisabledRest = this.disabledService.disabled(options.disabled, "请求中");
    this.http.request(options.url, options).subscribe(
      (response: Response) => {
        if (options.preventRepetitionId) {
          if (this.preventRepetitionRandomId[options.preventRepetitionId] !== options.preventRepetitionRandomId) {
            return;
          } else {
            delete this.preventRepetitionParamsRecord[options.preventRepetitionId];
            delete this.preventRepetitionRandomId[options.preventRepetitionId];
          }
        }

        /**
         * 成功
         */
        disabledRest.reset();
        if (options.alwaysDisabled) {
          this.disabledService.disabled(options.disabled, null, null);
        }
        this.success(response, options);
      },
      (response: Response) => {
        if (options.preventRepetitionId) {
          if (this.preventRepetitionRandomId[options.preventRepetitionId] !== options.preventRepetitionRandomId) {
            return;
          } else {
            delete this.preventRepetitionParamsRecord[options.preventRepetitionId];
            delete this.preventRepetitionRandomId[options.preventRepetitionId];
          }
        }
        /**
         * 失败
         */
        disabledRest.reset();
        this.error(response, options);
      }
    );
  }

  getData(response: Response, options: HttpOptionsArgs): any {
    let data: any;
    if (options.responseType === ResponseContentType.Json) {
      data = response.json();
    } else {
      data = response.text();
    }
    return data;
  }

  private defaultRequestOptionsArgs: RequestOptionsArgs = {
    headers: new Headers({
      "Content-Type": "application/json; charset=UTF-8",
      "X-Requested-With": "XMLHttpRequest"
    }),
    responseType: ResponseContentType.Json
  };

  private setDefaultOptions(options: HttpOptionsArgs) {
    for (let key in this.defaultRequestOptionsArgs) {
      if (key && this.toolsService.isNull((<any>options)[key])) {
        (<any>options)[key] = (<any>this.defaultRequestOptionsArgs)[key];
      }
    }
    options.url = this.toolsService.getFullUrl(this.baseUrl, options.url);
  }

  private success(response: Response, options: HttpOptionsArgs) {
    let data: any = this.getData(response, options);
    if (this.httpProvider.successIntercept(data, response, options)) {
      return;
    }
    if (options.success) {
      options.success(data, response, options);
    }
  }

  private error(response: Response, options: HttpOptionsArgs) {
    // 错误拦截之前 单个请求自己控制是否需要被拦截 或者在拦截之前do something 返回true则不继续往下走
    if (options.beforeErrorIntercept && options.beforeErrorIntercept(response, options)) {
      return;
    }
    // 错误拦截 各个业务系统可以根据自己的业务拦截统一的异常 返回true则不继续往下走
    if (this.httpProvider.errorIntercept(response, options)) {
      return;
    }
    if (options.error && options.error(response, options)) {
      return;
    }

    let message: string = "网络异常，请稍后再试！";
    let stackInformation: string = null;
    let errorCodeFlag: boolean = false;

    if (response.status === 500) {
      let data: {
        code: string;
        message: string;
        data: any;
        type: "BUSINESS" | "ERROR"
      } = this.getData(response, options);
      let code: string = response.status.toString();

      if (data && !isString(data)) {
        if (data.type === "BUSINESS") {
          code = data.code;
          message = data.message;
          if (options.errorCode) {
            if (isFunction(options.errorCode)) {
              (<ErrorCodeType>options.errorCode)(code, message, data.code, response, options);
              errorCodeFlag = true;
            } else if ((<ErrorCodeMapType>options.errorCode)[code]) {
              (<ErrorCodeMapType>options.errorCode)[code](message, data.code, response, options);
              errorCodeFlag = true;
            }
          }
        } else if (this.httpProvider.showStackInformation() && data.type === "ERROR") {
          stackInformation = data.message;
        }
      }
    }

    if (!errorCodeFlag && options.errorAlert !== false) {
      if (stackInformation) {
        this.alertService.open({
          iconType: Icon.error,
          content: message,
          yesLabel: "知道了",
          noLabel: "复制堆栈信息",
          noClick: () => {
            this.toolsService.copy(stackInformation);
          }
        });
      } else {
        this.alertService.open({
          iconType: Icon.error,
          content: message
        });
      }
    }
  }


  private urlParamsEncode(value: any) {
    if (value === null || value === undefined || value === "") {
      return "";
    }
    if (isDate(value)) {
      value = <Date>value.getTime();
    }
    value = (value + '').toString();
    return encodeURIComponent(value).replace(/!/g, '%21').replace(/'/g, '%27').replace(/\(/g, '%28').replace(/\)/g, '%29').replace(/\*/g, '%2A').replace(/%20/g, '+');
  }

  private urlParamsTranslate(params: {}): string {
    let str: string = '';
    for (let i in params) {
      if (i) {
        str += i + '=' + this.urlParamsEncode(params[i]) + '&';
      }
    }
    if (str) {
      str = str.substr(0, str.length - 1);
    }
    return str;
  }

}



