/**
 * 错误码定义
 */
import { SerialErrorCode, SerialError } from "./interface.uts"

/**
 * 错误主题
 */
export const UniErrorSubject = 'wzl-serialbridge';

/**
 * 错误信息映射
 */
export const SerialUniErrors: Map<SerialErrorCode, string> = new Map([
  [10001, '打开串口失败'],
  [10002, '写入数据失败'],
  [10003, '读取数据失败'],
  [10004, '关闭串口失败'],
  [10005, '串口未打开'],
  [10006, '参数错误'],
  [10007, '数据格式错误'],
]);

/**
 * 错误对象实现
 */
export class SerialErrorImpl extends UniError implements SerialError {
  override errCode: SerialErrorCode;

  constructor(errCode: SerialErrorCode) {
    super();
    this.errSubject = UniErrorSubject;
    this.errCode = errCode;
    this.errMsg = SerialUniErrors.get(errCode) ?? "";
  }
}
