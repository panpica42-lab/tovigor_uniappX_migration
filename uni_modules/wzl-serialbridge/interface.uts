/**
 * 串口通信插件 - TypeScript 接口定义
 * 
 * 基于 Kotlin SerialPort 库封装
 */

// ==================== 类型定义 ====================

/**
 * 数据位枚举
 */
export type DataBitsType = 5 | 6 | 7 | 8;

/**
 * 停止位枚举
 */
export type StopBitsType = 1 | 2;

/**
 * 校验位枚举
 */
export type ParityType = 'none' | 'odd' | 'even';

/**
 * 串口配置参数
 */
export type SerialConfig = {
  /** 波特率，默认 115200 */
  baudRate?: number;
  /** 数据位，默认 8 */
  dataBits?: DataBitsType;
  /** 停止位，默认 1 */
  stopBits?: StopBitsType;
  /** 校验位，默认 'none' */
  parity?: ParityType;
};

/**
 * 打开串口的参数
 */
export type OpenOptions = {
  /** 设备路径，如 "/dev/ttyS0" */
  path: string;
  /** 串口配置 */
  config?: SerialConfig;
  /** 成功回调 */
  success?: (res: OpenResult) => void;
  /** 失败回调 */
  fail?: (err: SerialError) => void;
  /** 完成回调 */
  complete?: (res: any) => void;
};

/**
 * 打开成功的返回值
 */
export type OpenResult = {
  errMsg: string;
  /** 串口句柄 ID */
  portId: number;
};

/**
 * 写数据参数
 */
export type WriteOptions = {
  /** 串口句柄 ID */
  portId: number;
  /** 要写入的数据 (HEX 字符串或 Base64) */
  data: string;
  /** 数据格式: 'hex' | 'base64' */
  format?: 'hex' | 'base64';
  /** 超时时间(毫秒)，默认 1000 */
  timeout?: number;
  /** 成功回调 */
  success?: (res: WriteResult) => void;
  /** 失败回调 */
  fail?: (err: SerialError) => void;
  /** 完成回调 */
  complete?: (res: any) => void;
};

/**
 * 写入成功的返回值
 */
export type WriteResult = {
  errMsg: string;
  /** 实际写入的字节数 */
  bytesWritten: number;
};

/**
 * 读数据参数
 */
export type ReadOptions = {
  /** 串口句柄 ID */
  portId: number;
  /** 期望读取的最大字节数，默认 1024 */
  length?: number;
  /** 返回数据格式: 'hex' | 'base64' */
  format?: 'hex' | 'base64';
  /** 超时时间(毫秒)，默认 1000 */
  timeout?: number;
  /** 成功回调 */
  success?: (res: ReadResult) => void;
  /** 失败回调 */
  fail?: (err: SerialError) => void;
  /** 完成回调 */
  complete?: (res: any) => void;
};

/**
 * 读取成功的返回值
 */
export type ReadResult = {
  errMsg: string;
  /** 读取到的数据 (HEX 或 Base64 格式) */
  data: string;
  /** 实际读取的字节数 */
  bytesRead: number;
};

/**
 * 关闭串口参数
 */
export type CloseOptions = {
  /** 串口句柄 ID */
  portId: number;
  /** 成功回调 */
  success?: (res: CloseResult) => void;
  /** 失败回调 */
  fail?: (err: SerialError) => void;
  /** 完成回调 */
  complete?: (res: any) => void;
};

/**
 * 关闭成功的返回值
 */
export type CloseResult = {
  errMsg: string;
};

/**
 * 列举设备参数
 */
export type ListDevicesOptions = {
  /** 设备路径前缀，默认 ["/dev/ttyS", "/dev/ttyUSB"] */
  prefixes?: string[];
  /** 成功回调 */
  success?: (res: ListDevicesResult) => void;
  /** 失败回调 */
  fail?: (err: SerialError) => void;
  /** 完成回调 */
  complete?: (res: any) => void;
};

/**
 * 列举设备返回值
 */
export type ListDevicesResult = {
  errMsg: string;
  /** 设备路径列表 */
  devices: string[];
};

// ==================== 错误码定义 ====================

/**
 * 错误码
 * - 10001: 打开串口失败
 * - 10002: 写入数据失败
 * - 10003: 读取数据失败
 * - 10004: 关闭串口失败
 * - 10005: 串口未打开
 * - 10006: 参数错误
 * - 10007: 数据格式错误
 */
export type SerialErrorCode = 10001 | 10002 | 10003 | 10004 | 10005 | 10006 | 10007;

/**
 * 错误对象
 */
export interface SerialError extends IUniError {
  errCode: SerialErrorCode;
}

// ==================== API 方法定义 ====================

/**
 * 打开串口
 */
export type OpenSerial = (options: OpenOptions) => void;

/**
 * 写入数据
 */
export type WriteSerial = (options: WriteOptions) => void;

/**
 * 读取数据
 */
export type ReadSerial = (options: ReadOptions) => void;

/**
 * 关闭串口
 */
export type CloseSerial = (options: CloseOptions) => void;

/**
 * 列举可用设备
 */
export type ListDevices = (options?: ListDevicesOptions) => void;

/**
 * 获取插件版本
 */
export type GetVersion = () => string;
