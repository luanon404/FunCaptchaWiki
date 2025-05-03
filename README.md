# Tài liệu Phân tích & Triển khai Fingerprint FunCaptcha

## Mục lục

- [Thông tin chung](#thông-tin-chung)
- [Dữ liệu Khởi tạo](#dữ-liệu-khởi-tạo)
- [Triển khai Hàm Hash](#triển-khai-hàm-hash)
- [Định dạng Khóa](#định-dạng-khóa)
- [Dữ liệu sinh trắc học](#dữ-liệu-sinh-trắc-học)
- [Dấu vân tay](#dấu-vân-tay)
  - [API Type](#api-type)
  - [Feature Values Hash](#feature-values-hash)
  - [Base64 Timestamp](#base64-timestamp)
  - [Window Hash](#window-hash)
  - [WebGL Rendering Context](#webgl-rendering-context)
  - [Enhanced Fingerprint](#enhanced-fingerprint)
  - [Features](#features)
  - [Feature Items Hash](#feature-items-hash)
  - [JavaScript Behavior Data](#javascript-behavior-data)

## Thông tin chung

|                          |                    |
| ------------------------ | ------------------ |
| Tác giả                  | `GnU-Chan`         |
| Cập nhật lần cuối        | `2024-03-27 03:00` |
| Định dạng ngày           | `yyyy-MM-dd HH:mm` |
| Seed mặc định MurmurHash | `0`                |
| Phiên bản                | `2.11.6`           |
| Trình duyệt              | `Chrome`           |
|                          |                    |

## Dữ liệu Khởi tạo

<details>
  <summary>Dữ liệu Khởi tạo</summary>

```javascript
const initData = {
	/**
	 * URL/href của trang hiện tại
	 * @type {string}
	 */
	chref: "https://demo.arkoselabs.com/",

	/**
	 * Mã ngôn ngữ hiện tại
	 * @type {string}
	 */
	clang: "en",

	/**
	 * URL dịch vụ cho các lệnh gọi API
	 * @type {string}
	 */
	surl: "https://client-api.arkoselabs.com",

	/**
	 * Có đang sử dụng chế độ SDK hay không
	 * @type {boolean}
	 */
	sdk: false,

	/**
	 * Cờ để phát hiện trình duyệt headless sử dụng Nightmare.js
	 * @type {boolean}
	 */
	nm: false,

	/**
	 * Có được kích hoạt inline hay không
	 * @type {boolean}
	 */
	triggeredInline: false,

	/**
	 * ID phiên
	 * @type {string}
	 * @identifier uuidv4
	 */
	"4b4b269e68": "3c7080a9-b3f7-420d-b6c0-50b3766edbb8",
};
```

</details>

## Triển khai Hàm Hash

<details>
  <summary>Triển khai Hàm Hash</summary>

```javascript
/**
 * Tập hợp các hàm băm được sử dụng cho việc tạo dấu vân tay
 * @namespace hashFunctions
 */
const hashFunctions = {
	/**
	 * Cộng hai số 64-bit được biểu diễn dưới dạng mảng
	 * @param {number[]} a - Số 64-bit thứ nhất dưới dạng mảng [high, low]
	 * @param {number[]} b - Số 64-bit thứ hai dưới dạng mảng [high, low]
	 * @returns {number[]} Tổng dưới dạng mảng [high, low]
	 */
	add64: function (a, b) {
		const a16 = [a[0] >>> 16, a[0] & 0xffff, a[1] >>> 16, a[1] & 0xffff];
		const b16 = [b[0] >>> 16, b[0] & 0xffff, b[1] >>> 16, b[1] & 0xffff];
		const result = [0, 0, 0, 0];

		result[3] += a16[3] + b16[3];
		result[2] += result[3] >>> 16;
		result[3] &= 0xffff;

		result[2] += a16[2] + b16[2];
		result[1] += result[2] >>> 16;
		result[2] &= 0xffff;

		result[1] += a16[1] + b16[1];
		result[0] += result[1] >>> 16;
		result[1] &= 0xffff;

		result[0] += a16[0] + b16[0];
		result[0] &= 0xffff;

		return [(result[0] << 16) | result[1], (result[2] << 16) | result[3]];
	},

	/**
	 * Nhân hai số 64-bit được biểu diễn dưới dạng mảng
	 * @param {number[]} a - Số 64-bit thứ nhất dưới dạng mảng [high, low]
	 * @param {number[]} b - Số 64-bit thứ hai dưới dạng mảng [high, low]
	 * @returns {number[]} Tích dưới dạng mảng [high, low]
	 */
	multiply64: function (a, b) {
		const a16 = [a[0] >>> 16, a[0] & 0xffff, a[1] >>> 16, a[1] & 0xffff];
		const b16 = [b[0] >>> 16, b[0] & 0xffff, b[1] >>> 16, b[1] & 0xffff];
		const result = [0, 0, 0, 0];

		result[3] += a16[3] * b16[3];
		result[2] += result[3] >>> 16;
		result[3] &= 0xffff;

		result[2] += a16[2] * b16[3];
		result[1] += result[2] >>> 16;
		result[2] &= 0xffff;

		result[2] += a16[3] * b16[2];
		result[1] += result[2] >>> 16;
		result[2] &= 0xffff;

		result[1] += a16[1] * b16[3];
		result[0] += result[1] >>> 16;
		result[1] &= 0xffff;

		result[1] += a16[2] * b16[2];
		result[0] += result[1] >>> 16;
		result[1] &= 0xffff;

		result[1] += a16[3] * b16[1];
		result[0] += result[1] >>> 16;
		result[1] &= 0xffff;

		result[0] += a16[0] * b16[3] + a16[1] * b16[2] + a16[2] * b16[1] + a16[3] * b16[0];
		result[0] &= 0xffff;

		return [(result[0] << 16) | result[1], (result[2] << 16) | result[3]];
	},

	/**
	 * Xoay trái một số 64-bit theo số bit chỉ định
	 * @param {number[]} num - Số 64-bit dưới dạng mảng [high, low]
	 * @param {number} bits - Số bit cần xoay
	 * @returns {number[]} Số đã xoay dưới dạng mảng [high, low]
	 */
	rotateLeft64: function (num, bits) {
		bits %= 64;
		if (bits === 32) return [num[1], num[0]];
		if (bits < 32) {
			return [(num[0] << bits) | (num[1] >>> (32 - bits)), (num[1] << bits) | (num[0] >>> (32 - bits))];
		}
		bits -= 32;
		return [(num[1] << bits) | (num[0] >>> (32 - bits)), (num[0] << bits) | (num[1] >>> (32 - bits))];
	},

	/**
	 * Dịch trái một số 64-bit theo số bit chỉ định
	 * @param {number[]} num - Số 64-bit dưới dạng mảng [high, low]
	 * @param {number} bits - Số bit cần dịch
	 * @returns {number[]} Số đã dịch dưới dạng mảng [high, low]
	 */
	shiftLeft64: function (num, bits) {
		bits %= 64;
		if (bits === 0) return num;
		if (bits < 32) {
			return [(num[0] << bits) | (num[1] >>> (32 - bits)), num[1] << bits];
		}
		return [num[1] << (bits - 32), 0];
	},

	/**
	 * XOR hai số 64-bit
	 * @param {number[]} a - Số 64-bit thứ nhất dưới dạng mảng [high, low]
	 * @param {number[]} b - Số 64-bit thứ hai dưới dạng mảng [high, low]
	 * @returns {number[]} Kết quả XOR dưới dạng mảng [high, low]
	 */
	xor64: function (a, b) {
		return [a[0] ^ b[0], a[1] ^ b[1]];
	},

	/**
	 * Hoàn thiện giá trị băm 64-bit
	 * @param {number[]} hash - Giá trị băm 64-bit dưới dạng mảng [high, low]
	 * @returns {number[]} Giá trị băm đã hoàn thiện dưới dạng mảng [high, low]
	 */
	finalize64: function (hash) {
		hash = this.xor64(hash, [0, hash[0] >>> 1]);
		hash = this.multiply64(hash, [4283543511, 3981806797]);
		hash = this.xor64(hash, [0, hash[0] >>> 1]);
		hash = this.multiply64(hash, [3301882366, 444984403]);
		hash = this.xor64(hash, [0, hash[0] >>> 1]);
		return hash;
	},

	/**
	 * Triển khai thuật toán MurmurHash3
	 * @param {string} key - Chuỗi đầu vào cần băm
	 * @param {number} [seed=0] - Giá trị seed tùy chọn
	 * @returns {string} Giá trị băm 128-bit dưới dạng chuỗi hex
	 */
	murmurHash3: function (key, seed = 0) {
		if (!key) return "";

		const len = key.length;
		const remainder = len % 16;
		const blocks = len - remainder;

		let hash1 = [0, seed];
		let hash2 = [0, seed];
		let k1 = [0, 0];
		let k2 = [0, 0];
		const c1 = [2277735313, 289559509];
		const c2 = [1291169091, 658871167];

		// Xử lý các khối 16-byte
		for (let i = 0; i < blocks; i += 16) {
			k1 = [(key.charCodeAt(i + 4) & 0xff) | ((key.charCodeAt(i + 5) & 0xff) << 8) | ((key.charCodeAt(i + 6) & 0xff) << 16) | ((key.charCodeAt(i + 7) & 0xff) << 24), (key.charCodeAt(i) & 0xff) | ((key.charCodeAt(i + 1) & 0xff) << 8) | ((key.charCodeAt(i + 2) & 0xff) << 16) | ((key.charCodeAt(i + 3) & 0xff) << 24)];

			k2 = [(key.charCodeAt(i + 12) & 0xff) | ((key.charCodeAt(i + 13) & 0xff) << 8) | ((key.charCodeAt(i + 14) & 0xff) << 16) | ((key.charCodeAt(i + 15) & 0xff) << 24), (key.charCodeAt(i + 8) & 0xff) | ((key.charCodeAt(i + 9) & 0xff) << 8) | ((key.charCodeAt(i + 10) & 0xff) << 16) | ((key.charCodeAt(i + 11) & 0xff) << 24)];

			k1 = this.multiply64(k1, c1);
			k1 = this.rotateLeft64(k1, 31);
			k1 = this.multiply64(k1, c2);
			hash1 = this.xor64(hash1, k1);

			hash1 = this.rotateLeft64(hash1, 27);
			hash1 = this.add64(hash1, hash2);
			hash1 = this.add64(this.multiply64(hash1, [0, 5]), [0, 1390208809]);

			k2 = this.multiply64(k2, c2);
			k2 = this.rotateLeft64(k2, 33);
			k2 = this.multiply64(k2, c1);
			hash2 = this.xor64(hash2, k2);

			hash2 = this.rotateLeft64(hash2, 31);
			hash2 = this.add64(hash2, hash1);
			hash2 = this.add64(this.multiply64(hash2, [0, 5]), [0, 944331445]);
		}

		// Xử lý các byte còn lại
		k1 = [0, 0];
		k2 = [0, 0];

		switch (remainder) {
			case 15:
				k2 = this.xor64(k2, this.shiftLeft64([0, key.charCodeAt(blocks + 14)], 48));
			case 14:
				k2 = this.xor64(k2, this.shiftLeft64([0, key.charCodeAt(blocks + 13)], 40));
			case 13:
				k2 = this.xor64(k2, this.shiftLeft64([0, key.charCodeAt(blocks + 12)], 32));
			case 12:
				k2 = this.xor64(k2, this.shiftLeft64([0, key.charCodeAt(blocks + 11)], 24));
			case 11:
				k2 = this.xor64(k2, this.shiftLeft64([0, key.charCodeAt(blocks + 10)], 16));
			case 10:
				k2 = this.xor64(k2, this.shiftLeft64([0, key.charCodeAt(blocks + 9)], 8));
			case 9:
				k2 = this.xor64(k2, [0, key.charCodeAt(blocks + 8)]);
				k2 = this.multiply64(k2, c2);
				k2 = this.rotateLeft64(k2, 33);
				k2 = this.multiply64(k2, c1);
				hash2 = this.xor64(hash2, k2);
			case 8:
				k1 = this.xor64(k1, this.shiftLeft64([0, key.charCodeAt(blocks + 7)], 56));
			case 7:
				k1 = this.xor64(k1, this.shiftLeft64([0, key.charCodeAt(blocks + 6)], 48));
			case 6:
				k1 = this.xor64(k1, this.shiftLeft64([0, key.charCodeAt(blocks + 5)], 40));
			case 5:
				k1 = this.xor64(k1, this.shiftLeft64([0, key.charCodeAt(blocks + 4)], 32));
			case 4:
				k1 = this.xor64(k1, this.shiftLeft64([0, key.charCodeAt(blocks + 3)], 24));
			case 3:
				k1 = this.xor64(k1, this.shiftLeft64([0, key.charCodeAt(blocks + 2)], 16));
			case 2:
				k1 = this.xor64(k1, this.shiftLeft64([0, key.charCodeAt(blocks + 1)], 8));
			case 1:
				k1 = this.xor64(k1, [0, key.charCodeAt(blocks)]);
				k1 = this.multiply64(k1, c1);
				k1 = this.rotateLeft64(k1, 31);
				k1 = this.multiply64(k1, c2);
				hash1 = this.xor64(hash1, k1);
		}

		hash1 = this.xor64(hash1, [0, len]);
		hash2 = this.xor64(hash2, [0, len]);

		hash1 = this.add64(hash1, hash2);
		hash2 = this.add64(hash2, hash1);

		hash1 = this.finalize64(hash1);
		hash2 = this.finalize64(hash2);

		hash1 = this.add64(hash1, hash2);
		hash2 = this.add64(hash2, hash1);

		return ("00000000" + (hash1[0] >>> 0).toString(16)).slice(-8) + ("00000000" + (hash1[1] >>> 0).toString(16)).slice(-8) + ("00000000" + (hash2[0] >>> 0).toString(16)).slice(-8) + ("00000000" + (hash2[1] >>> 0).toString(16)).slice(-8);
	},

	/**
	 * Hàm băm chuỗi đơn giản
	 * @param {string} str - Chuỗi đầu vào cần băm
	 * @returns {string|number} Giá trị băm
	 */
	simpleHash: function (str) {
		if (!str) return "";

		if (Array.prototype.reduce) {
			return str.split("").reduce((hash, char) => {
				hash = (hash << 5) - hash + char.charCodeAt(0);
				return hash & hash;
			}, 0);
		}

		let hash = 0;
		if (str.length === 0) return hash;

		for (let i = 0; i < str.length; i++) {
			hash = (hash << 5) - hash + str.charCodeAt(i);
			hash = hash & hash;
		}
		return hash;
	},

	/**
	 * Triển khai MD5 trong JavaScript
	 * Dựa trên https://github.com/blueimp/JavaScript-MD5
	 *
	 * Bản quyền gốc:
	 * Copyright 2011, Sebastian Tschan
	 * https://blueimp.net
	 * Được cấp phép theo MIT
	 *
	 * Dựa trên thuật toán MD5 Message Digest của RSA Data Security, Inc. (RFC 1321)
	 * Phiên bản 2.2 Copyright (C) Paul Johnston 1999-2009
	 * Các đóng góp khác: Greg Holt, Andrew Kepert, Ydnar, Lostinet
	 */

	/**
	 * Cộng hai số 32-bit sử dụng các phép toán bit
	 * @param {number} num1 - Số 32-bit thứ nhất cần cộng
	 * @param {number} num2 - Số 32-bit thứ hai cần cộng
	 * @returns {number} Tổng của hai số dưới dạng số nguyên 32-bit
	 */
	addNumbers: function (num1, num2) {
		var lowerBits = (num1 & 0xffff) + (num2 & 0xffff);
		var upperBits = (num1 >> 16) + (num2 >> 16) + (lowerBits >> 16);
		return (upperBits << 16) | (lowerBits & 0xffff);
	},

	/**
	 * Xoay trái một số 32-bit
	 * @param {number} number - Số cần xoay
	 * @param {number} positions - Số vị trí cần xoay
	 * @returns {number} Số đã xoay
	 */
	rotateLeft: function (number, positions) {
		return (number << positions) | (number >>> (32 - positions));
	},

	/**
	 * Thực hiện phép biến đổi MD5
	 * @param {number} operation - Giá trị phép toán
	 * @param {number} val1 - Giá trị thứ nhất
	 * @param {number} val2 - Giá trị thứ hai
	 * @param {number} val3 - Giá trị thứ ba
	 * @param {number} shiftAmount - Số lượng dịch bit
	 * @param {number} addConst - Hằng số cần cộng thêm
	 * @returns {number} Giá trị đã biến đổi
	 */
	md5Transform: function (operation, val1, val2, val3, shiftAmount, addConst) {
		return this.addNumbers(this.rotateLeft(this.addNumbers(this.addNumbers(val1, operation), this.addNumbers(val3, addConst)), shiftAmount), val2);
	},

	/**
	 * Thực hiện phép toán vòng 1 của MD5
	 * @param {number} val1 - Giá trị thứ nhất
	 * @param {number} val2 - Giá trị thứ hai
	 * @param {number} val3 - Giá trị thứ ba
	 * @param {number} val4 - Giá trị thứ tư
	 * @param {number} blockData - Dữ liệu khối
	 * @param {number} shiftAmount - Số lượng dịch bit
	 * @param {number} addConst - Hằng số cần cộng thêm
	 * @returns {number} Kết quả vòng 1
	 */
	md5Round1: function (val1, val2, val3, val4, blockData, shiftAmount, addConst) {
		return this.md5Transform((val2 & val3) | (~val2 & val4), val1, val2, blockData, shiftAmount, addConst);
	},

	/**
	 * Thực hiện phép toán vòng 2 của MD5
	 * @param {number} val1 - Giá trị thứ nhất
	 * @param {number} val2 - Giá trị thứ hai
	 * @param {number} val3 - Giá trị thứ ba
	 * @param {number} val4 - Giá trị thứ tư
	 * @param {number} blockData - Dữ liệu khối
	 * @param {number} shiftAmount - Số lượng dịch bit
	 * @param {number} addConst - Hằng số cần cộng thêm
	 * @returns {number} Kết quả vòng 2
	 */
	md5Round2: function (val1, val2, val3, val4, blockData, shiftAmount, addConst) {
		return this.md5Transform((val2 & val4) | (val3 & ~val4), val1, val2, blockData, shiftAmount, addConst);
	},

	/**
	 * Thực hiện phép toán vòng 3 của MD5
	 * @param {number} val1 - Giá trị thứ nhất
	 * @param {number} val2 - Giá trị thứ hai
	 * @param {number} val3 - Giá trị thứ ba
	 * @param {number} val4 - Giá trị thứ tư
	 * @param {number} blockData - Dữ liệu khối
	 * @param {number} shiftAmount - Số lượng dịch bit
	 * @param {number} addConst - Hằng số cần cộng thêm
	 * @returns {number} Kết quả vòng 3
	 */
	md5Round3: function (val1, val2, val3, val4, blockData, shiftAmount, addConst) {
		return this.md5Transform(val2 ^ val3 ^ val4, val1, val2, blockData, shiftAmount, addConst);
	},

	/**
	 * Thực hiện phép toán vòng 4 của MD5
	 * @param {number} val1 - Giá trị thứ nhất
	 * @param {number} val2 - Giá trị thứ hai
	 * @param {number} val3 - Giá trị thứ ba
	 * @param {number} val4 - Giá trị thứ tư
	 * @param {number} blockData - Dữ liệu khối
	 * @param {number} shiftAmount - Số lượng dịch bit
	 * @param {number} addConst - Hằng số cần cộng thêm
	 * @returns {number} Kết quả vòng 4
	 */
	md5Round4: function (val1, val2, val3, val4, blockData, shiftAmount, addConst) {
		return this.md5Transform(val3 ^ (val2 | ~val4), val1, val2, blockData, shiftAmount, addConst);
	},

	/**
	 * Tính toán các khối MD5
	 * @param {number[]} blocks - Các khối đầu vào
	 * @param {number} dataLength - Độ dài của dữ liệu
	 * @returns {number[]} Các khối MD5
	 */
	calculateMD5Blocks: function (blocks, dataLength) {
		blocks[dataLength >> 5] |= 0x80 << dataLength % 32;
		blocks[(((dataLength + 64) >>> 9) << 4) + 14] = dataLength;

		var i;
		var previousA;
		var previousB;
		var previousC;
		var previousD;
		var hashA = 1732584193;
		var hashB = -271733879;
		var hashC = -1732584194;
		var hashD = 271733878;

		for (i = 0; i < blocks.length; i += 16) {
			previousA = hashA;
			previousB = hashB;
			previousC = hashC;
			previousD = hashD;

			hashA = this.md5Round1(hashA, hashB, hashC, hashD, blocks[i], 7, -680876936);
			hashD = this.md5Round1(hashD, hashA, hashB, hashC, blocks[i + 1], 12, -389564586);
			hashC = this.md5Round1(hashC, hashD, hashA, hashB, blocks[i + 2], 17, 606105819);
			hashB = this.md5Round1(hashB, hashC, hashD, hashA, blocks[i + 3], 22, -1044525330);
			hashA = this.md5Round1(hashA, hashB, hashC, hashD, blocks[i + 4], 7, -176418897);
			hashD = this.md5Round1(hashD, hashA, hashB, hashC, blocks[i + 5], 12, 1200080426);
			hashC = this.md5Round1(hashC, hashD, hashA, hashB, blocks[i + 6], 17, -1473231341);
			hashB = this.md5Round1(hashB, hashC, hashD, hashA, blocks[i + 7], 22, -45705983);
			hashA = this.md5Round1(hashA, hashB, hashC, hashD, blocks[i + 8], 7, 1770035416);
			hashD = this.md5Round1(hashD, hashA, hashB, hashC, blocks[i + 9], 12, -1958414417);
			hashC = this.md5Round1(hashC, hashD, hashA, hashB, blocks[i + 10], 17, -42063);
			hashB = this.md5Round1(hashB, hashC, hashD, hashA, blocks[i + 11], 22, -1990404162);
			hashA = this.md5Round1(hashA, hashB, hashC, hashD, blocks[i + 12], 7, 1804603682);
			hashD = this.md5Round1(hashD, hashA, hashB, hashC, blocks[i + 13], 12, -40341101);
			hashC = this.md5Round1(hashC, hashD, hashA, hashB, blocks[i + 14], 17, -1502002290);
			hashB = this.md5Round1(hashB, hashC, hashD, hashA, blocks[i + 15], 22, 1236535329);

			hashA = this.md5Round2(hashA, hashB, hashC, hashD, blocks[i + 1], 5, -165796510);
			hashD = this.md5Round2(hashD, hashA, hashB, hashC, blocks[i + 6], 9, -1069501632);
			hashC = this.md5Round2(hashC, hashD, hashA, hashB, blocks[i + 11], 14, 643717713);
			hashB = this.md5Round2(hashB, hashC, hashD, hashA, blocks[i], 20, -373897302);
			hashA = this.md5Round2(hashA, hashB, hashC, hashD, blocks[i + 5], 5, -701558691);
			hashD = this.md5Round2(hashD, hashA, hashB, hashC, blocks[i + 10], 9, 38016083);
			hashC = this.md5Round2(hashC, hashD, hashA, hashB, blocks[i + 15], 14, -660478335);
			hashB = this.md5Round2(hashB, hashC, hashD, hashA, blocks[i + 4], 20, -405537848);
			hashA = this.md5Round2(hashA, hashB, hashC, hashD, blocks[i + 9], 5, 568446438);
			hashD = this.md5Round2(hashD, hashA, hashB, hashC, blocks[i + 14], 9, -1019803690);
			hashC = this.md5Round2(hashC, hashD, hashA, hashB, blocks[i + 3], 14, -187363961);
			hashB = this.md5Round2(hashB, hashC, hashD, hashA, blocks[i + 8], 20, 1163531501);
			hashA = this.md5Round2(hashA, hashB, hashC, hashD, blocks[i + 13], 5, -1444681467);
			hashD = this.md5Round2(hashD, hashA, hashB, hashC, blocks[i + 2], 9, -51403784);
			hashC = this.md5Round2(hashC, hashD, hashA, hashB, blocks[i + 7], 14, 1735328473);
			hashB = this.md5Round2(hashB, hashC, hashD, hashA, blocks[i + 12], 20, -1926607734);

			hashA = this.md5Round3(hashA, hashB, hashC, hashD, blocks[i + 5], 4, -378558);
			hashD = this.md5Round3(hashD, hashA, hashB, hashC, blocks[i + 8], 11, -2022574463);
			hashC = this.md5Round3(hashC, hashD, hashA, hashB, blocks[i + 11], 16, 1839030562);
			hashB = this.md5Round3(hashB, hashC, hashD, hashA, blocks[i + 14], 23, -35309556);
			hashA = this.md5Round3(hashA, hashB, hashC, hashD, blocks[i + 1], 4, -1530992060);
			hashD = this.md5Round3(hashD, hashA, hashB, hashC, blocks[i + 4], 11, 1272893353);
			hashC = this.md5Round3(hashC, hashD, hashA, hashB, blocks[i + 7], 16, -155497632);
			hashB = this.md5Round3(hashB, hashC, hashD, hashA, blocks[i + 10], 23, -1094730640);
			hashA = this.md5Round3(hashA, hashB, hashC, hashD, blocks[i + 13], 4, 681279174);
			hashD = this.md5Round3(hashD, hashA, hashB, hashC, blocks[i], 11, -358537222);
			hashC = this.md5Round3(hashC, hashD, hashA, hashB, blocks[i + 3], 16, -722521979);
			hashB = this.md5Round3(hashB, hashC, hashD, hashA, blocks[i + 6], 23, 76029189);
			hashA = this.md5Round3(hashA, hashB, hashC, hashD, blocks[i + 9], 4, -640364487);
			hashD = this.md5Round3(hashD, hashA, hashB, hashC, blocks[i + 12], 11, -421815835);
			hashC = this.md5Round3(hashC, hashD, hashA, hashB, blocks[i + 15], 16, 530742520);
			hashB = this.md5Round3(hashB, hashC, hashD, hashA, blocks[i + 2], 23, -995338651);

			hashA = this.md5Round4(hashA, hashB, hashC, hashD, blocks[i], 6, -198630844);
			hashD = this.md5Round4(hashD, hashA, hashB, hashC, blocks[i + 7], 10, 1126891415);
			hashC = this.md5Round4(hashC, hashD, hashA, hashB, blocks[i + 14], 15, -1416354905);
			hashB = this.md5Round4(hashB, hashC, hashD, hashA, blocks[i + 5], 21, -57434055);
			hashA = this.md5Round4(hashA, hashB, hashC, hashD, blocks[i + 12], 6, 1700485571);
			hashD = this.md5Round4(hashD, hashA, hashB, hashC, blocks[i + 3], 10, -1894986606);
			hashC = this.md5Round4(hashC, hashD, hashA, hashB, blocks[i + 10], 15, -1051523);
			hashB = this.md5Round4(hashB, hashC, hashD, hashA, blocks[i + 1], 21, -2054922799);
			hashA = this.md5Round4(hashA, hashB, hashC, hashD, blocks[i + 8], 6, 1873313359);
			hashD = this.md5Round4(hashD, hashA, hashB, hashC, blocks[i + 15], 10, -30611744);
			hashC = this.md5Round4(hashC, hashD, hashA, hashB, blocks[i + 6], 15, -1560198380);
			hashB = this.md5Round4(hashB, hashC, hashD, hashA, blocks[i + 13], 21, 1309151649);
			hashA = this.md5Round4(hashA, hashB, hashC, hashD, blocks[i + 4], 6, -145523070);
			hashD = this.md5Round4(hashD, hashA, hashB, hashC, blocks[i + 11], 10, -1120210379);
			hashC = this.md5Round4(hashC, hashD, hashA, hashB, blocks[i + 2], 15, 718787259);
			hashB = this.md5Round4(hashB, hashC, hashD, hashA, blocks[i + 9], 21, -343485551);

			hashA = this.addNumbers(hashA, previousA);
			hashB = this.addNumbers(hashB, previousB);
			hashC = this.addNumbers(hashC, previousC);
			hashD = this.addNumbers(hashD, previousD);
		}
		return [hashA, hashB, hashC, hashD];
	},

	/**
	 * Chuyển đổi mảng các khối 32-bit thành chuỗi
	 * @param {number[]} blocks - Mảng các khối 32-bit
	 * @returns {string} Chuỗi kết quả
	 */
	blocksToString: function (blocks) {
		var result = "";
		var totalBits = blocks.length * 32;
		for (var i = 0; i < totalBits; i += 8) {
			result += String.fromCharCode((blocks[i >> 5] >>> i % 32) & 0xff);
		}
		return result;
	},

	/**
	 * Chuyển đổi chuỗi thành mảng các khối 32-bit
	 * @param {string} input - Chuỗi đầu vào cần chuyển đổi
	 * @returns {number[]} Mảng các khối 32-bit
	 */
	stringToBlocks: function (input) {
		var blocks = [];
		blocks[(input.length >> 2) - 1] = undefined;
		for (var i = 0; i < blocks.length; i += 1) {
			blocks[i] = 0;
		}
		var totalBits = input.length * 8;
		for (var i = 0; i < totalBits; i += 8) {
			blocks[i >> 5] |= (input.charCodeAt(i / 8) & 0xff) << i % 32;
		}
		return blocks;
	},

	/**
	 * Tính toán giá trị băm MD5 của một chuỗi
	 * @param {string} str - Chuỗi đầu vào
	 * @returns {string} Giá trị băm MD5
	 */
	calculateMD5: function (str) {
		return this.blocksToString(this.calculateMD5Blocks(this.stringToBlocks(str), str.length * 8));
	},

	/**
	 * Tính toán giá trị băm HMAC-MD5
	 * @param {string} key - Khóa HMAC
	 * @param {string} data - Dữ liệu đầu vào
	 * @returns {string} Giá trị băm HMAC-MD5
	 */
	calculateHMACMD5: function (key, data) {
		var keyBlocks = this.stringToBlocks(key);
		var innerPadding = [];
		var outerPadding = [];
		var hash;
		innerPadding[15] = outerPadding[15] = undefined;
		if (keyBlocks.length > 16) {
			keyBlocks = this.calculateMD5Blocks(keyBlocks, key.length * 8);
		}
		for (var i = 0; i < 16; i += 1) {
			innerPadding[i] = keyBlocks[i] ^ 0x36363636;
			outerPadding[i] = keyBlocks[i] ^ 0x5c5c5c5c;
		}
		hash = this.calculateMD5Blocks(innerPadding.concat(this.stringToBlocks(data)), 512 + data.length * 8);
		return this.blocksToString(this.calculateMD5Blocks(outerPadding.concat(hash), 512 + 128));
	},

	/**
	 * Chuyển đổi chuỗi sang dạng thập lục phân
	 * @param {string} input - Chuỗi đầu vào
	 * @returns {string} Chuỗi thập lục phân
	 */
	stringToHex: function (input) {
		var hexDigits = "0123456789abcdef";
		var output = "";
		for (var i = 0; i < input.length; i += 1) {
			var charCode = input.charCodeAt(i);
			output += hexDigits.charAt((charCode >>> 4) & 0x0f) + hexDigits.charAt(charCode & 0x0f);
		}
		return output;
	},

	/**
	 * Chuyển đổi chuỗi sang UTF-8
	 * @param {string} input - Chuỗi đầu vào
	 * @returns {string} Chuỗi đã mã hóa UTF-8
	 */
	stringToUTF8: function (input) {
		return unescape(encodeURIComponent(input));
	},

	/**
	 * Tính toán giá trị băm MD5 thô của một chuỗi
	 * @param {string} str - Chuỗi đầu vào
	 * @returns {string} Giá trị băm MD5 thô
	 */
	calculateRawMD5: function (str) {
		return this.calculateMD5(this.stringToUTF8(str));
	},

	/**
	 * Tính toán giá trị băm MD5 dạng hex của một chuỗi
	 * @param {string} str - Chuỗi đầu vào
	 * @returns {string} Giá trị băm MD5 dạng hex
	 */
	calculateHexMD5: function (str) {
		return this.stringToHex(this.calculateRawMD5(str));
	},

	/**
	 * Tính toán giá trị băm HMAC-MD5 thô
	 * @param {string} key - Khóa HMAC
	 * @param {string} data - Dữ liệu đầu vào
	 * @returns {string} Giá trị băm HMAC-MD5 thô
	 */
	calculateRawHMACMD5: function (key, data) {
		return this.calculateHMACMD5(this.stringToUTF8(key), this.stringToUTF8(data));
	},

	/**
	 * Tính toán giá trị băm HMAC-MD5 dạng hex
	 * @param {string} key - Khóa HMAC
	 * @param {string} data - Dữ liệu đầu vào
	 * @returns {string} Giá trị băm HMAC-MD5 dạng hex
	 */
	calculateHexHMACMD5: function (key, data) {
		return this.stringToHex(this.calculateRawHMACMD5(key, data));
	},

	/**
	 * Hàm băm MD5 chính
	 * @param {string} input - Chuỗi đầu vào cần băm
	 * @param {string} [secretKey] - Khóa HMAC tùy chọn
	 * @param {boolean} [returnRaw] - Có trả về giá trị băm thô hay không
	 * @returns {string} Giá trị băm MD5
	 */
	md5: function (input, secretKey, returnRaw) {
		if (!secretKey) {
			if (!returnRaw) {
				return this.calculateHexMD5(input);
			}
			return this.calculateRawMD5(input);
		}
		if (!returnRaw) {
			return this.calculateHexHMACMD5(secretKey, input);
		}
		return this.calculateRawHMACMD5(secretKey, input);
	},
};
```

</details>

## Định dạng Khóa

````markdown
### Tên

- ID: `ID`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  Code;
  ```

  </details>

- Ví dụ: `Example`
````

HOẶC

````markdown
#### Tên Con

- ID: `ID`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  Code;
  ```

  </details>

- Ví dụ: `Example`
````

## Dữ liệu sinh trắc học

### Phần 1: Khởi tạo

- ID: `mbio,tbio,kbio`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
    const biometrics = {
      mouseEvents: [],
      touchEvents: [],
      keyPressEvents: [],
      timestamp: Date.now(),
      _listenersSetup: false,
      _mouseEventsEnabled: true,
      _touchEventsEnabled: true,
      _keyPressEventsEnabled: true,

      resetBiometrics: function () {
        this.mouseEvents = [];
        this.touchEvents = [];
        this.keyPressEvents = [];
        this.timestamp = Date.now();
      },
  ```

  </details>

- Ví dụ: `None`

### Phần 2: Các hàm thu thập dữ liệu

- ID: `None`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
      mouse: {
        // Các loại sự kiện được chuyển thành số
        // mousemove -> 0
        // mousedown -> 1
        // mouseup -> 2
        const EVENT_TYPES = {
          MOVE: 0,
          DOWN: 1,
          UP: 2
        };
        record: function (eventType) {
          // Sử dụng biến parent để tham chiếu đến đối tượng biometrics
          const parent = biometrics;
          return function (event) {
            function recordEvent() {
              const newEvent = {
                timestamp: Date.now() - parent.timestamp,
                type: eventType,
                x: event.pageX,
                y: event.pageY
              };
              parent.mouseEvents.push(newEvent);
              parent._lastMouseMove = newEvent;
            }

            // Giới hạn số lượng sự kiện chuột là 150
            if (parent.mouseEvents.length > 150) return;

            if (eventType === "mousemove") {
              if (parent._lastMouseMove) {
                const dx = event.pageX - parent._lastMouseMove.x;
                const dy = event.pageY - parent._lastMouseMove.y;
                const distance = Math.sqrt(dx * dx + dy * dy);
                if (distance > 5) {
                  recordEvent();
                }
              } else {
                recordEvent();
              }
              return;
            }

            // Ghi nhận các loại sự kiện chuột khác (mousedown, mouseup)
            parent.mouseEvents.push({
              timestamp: Date.now() - parent.timestamp,
              type: eventType,
              x: event.pageX,
              y: event.pageY
            });
          };
        }
      },

      touch: {
        // Các loại sự kiện được chuyển thành số
        // touchstart -> 0
        // touchend -> 1
        // touchmove -> 2
        // touchcancel -> 99
        const EVENT_TYPES = {
          START: 0,
          END: 1,
          MOVE: 2,
          CANCEL: 99
        };
        record: function (eventType) {
          const parent = biometrics;
          return function (touchEvent) {
            for (let i = 0; i < touchEvent.touches.length; i++) {
              if (parent.touchEvents.length < 150) {
                parent.touchEvents.push({
                  timestamp: Date.now() - parent.timestamp,
                  type: eventType,
                  x: Math.floor(touchEvent.touches[i].pageX),
                  y: Math.floor(touchEvent.touches[i].pageY)
                });
              }
            }
          };
        }
      },

      keyboard: {
        // Các loại sự kiện được chuyển thành số
        // keydown -> 0
        // keyup -> 1
        const EVENT_TYPES = {
          DOWN: 0,
          UP: 1
        };
        record: function (eventType) {
          const parent = biometrics;
          const keyMapping = {
            Tab: 0,
            Enter: 1,
            Space: 3,
            ShiftLeft: 4,
            ShiftRight: 5,
            ControlLeft: 6,
            ControlRight: 7,
            MetaLeft: 8,
            MetaRight: 9,
            AltLeft: 10,
            AltRight: 11,
            Backspace: 12,
            Escape: 13
          };
          return function (keyEvent) {
            if (parent.keyPressEvents.length < 50) {
              parent.keyPressEvents.push({
                timestamp: Date.now() - parent.timestamp,
                type: eventType,
                code: keyMapping[keyEvent.code] !== undefined ? keyMapping[keyEvent.code] : 14
              });
            }
          };
        }
      },
  ```

  </details>

- Ví dụ: `None`

### Phần 3: Các hàm xử lý dữ liệu

- ID: `None`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
      // Hàm chuyển đổi dữ liệu sinh trắc học sang định dạng JSON rồi mã hóa Base64
      getBiometricsData: function () {
        const data = {
          mouse: this.mouseEvents.map(e => `${e.timestamp},${e.type},${e.x},${e.y}`).join(";"),
          touch: this.touchEvents.map(e => `${e.timestamp},${e.type},${e.x},${e.y}`).join(";"),
          keyPress: this.keyPressEvents.map(e => `${e.timestamp},${e.type},${e.code}`).join(";")
        };
        return window.btoa(JSON.stringify(data));
      },

      // Thiết lập các trình lắng nghe sự kiện nếu chúng chưa được cấu hình
      setupListeners: function () {
        if (this._listenersSetup) return;
        this._listenersSetup = true;

        if (this._mouseEventsEnabled) {
          document.addEventListener("mousemove", this.mouse.record("mousemove"));
          document.addEventListener("mousedown", this.mouse.record("mousedown"));
          document.addEventListener("mouseup", this.mouse.record("mouseup"));
        }
        if (this._touchEventsEnabled) {
          document.addEventListener("touchstart", this.touch.record("touchstart"), { passive: false });
          document.addEventListener("touchend", this.touch.record("touchend"), { passive: false });
          document.addEventListener("touchmove", this.touch.record("touchmove"), { passive: false });
          document.addEventListener("touchcancel", this.touch.record("touchcancel"), { passive: false });
        }
        if (this._keyPressEventsEnabled) {
          document.addEventListener("keydown", this.keyboard.record("keydown"));
          document.addEventListener("keyup", this.keyboard.record("keyup"));
        }
      }
    };

    // Khởi tạo các trình lắng nghe sự kiện khi tài liệu đã sẵn sàng
    biometrics.setupListeners();

    // Cho phép truy cập đối tượng biometrics toàn cục để dễ dàng kiểm thử và tùy chỉnh
    window.biometrics = biometrics;
  })();
  ```

  </details>

- Ví dụ: `None`

## Dấu vân tay

### API Type

- ID: `api_type`
- Mô tả: Kiểu API được sử dụng, luôn có giá trị là "js"
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  "js";
  ```

  </details>

- Ví dụ: `"js"`

### Feature Values Hash

- ID: `f`
- Mô tả: Băm các giá trị tính năng của trình duyệt bằng thuật toán MurmurHash3
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  hashFunctions.murmurHash3(fe_values.join(";"), 0);
  ```

  </details>

- Ví dụ: `"d3a9ef4126937abea72ae1be3554df64"`

### Base64 Timestamp

- ID: `n`
- Mô tả: Thời gian hiện tại (Unix Timestamp) được mã hóa Base64
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  btoa(Math.floor(Date.now() / 1000).toString());
  ```

  </details>

- Ví dụ: `"MTczNTgxNTA5MA=="` // Giải mã: 1735815090 (Unix timestamp)

### Window Hash

- ID: `wh`
- Mô tả: Băm các thuộc tính và chuỗi prototype của đối tượng window bằng thuật toán MurmurHash3
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  // Lấy danh sách các thuộc tính của window
  function getWindowProperties() {
  	if (!Object.getOwnPropertyNames) return "LEGACY_ENV";
  	const patterns = ["f_", "arkoseLabsClientApi", "webpack"];
  	const regex = new RegExp("^(" + patterns.join("|") + ").*");
  	const properties = Object.getOwnPropertyNames(window).filter((propertyName) => !propertyName.match(regex));
  	return properties.sort().join("|");
  }

  // Lấy chuỗi prototype của window
  function getWindowProtoChain() {
  	if (!Object.getOwnPropertyNames) return "LEGACY_ENV";
  	const properties = [];
  	let proto = window;
  	while (Object.getPrototypeOf(proto)) {
  		proto = Object.getPrototypeOf(proto);
  		properties.push(...Object.getOwnPropertyNames(proto));
  	}
  	return properties.join("|");
  }

  // Nối và băm các giá trị
  "".concat(hashFunctions.murmurHash3(getWindowProperties(), 420), "|", hashFunctions.murmurHash3(getWindowProtoChain(), 420));
  ```

  </details>

- Ví dụ: `"7a6b6d9e6fd8d28e2356df7d8f577aa0|72627afbfd19a741c7da1732218301ac"`

### WebGL Rendering Context

- ID: `null`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  const canvas = document.createElement("canvas");
  const webglContext = canvas.getContext("webgl") || canvas.getContext("experimental-webgl");
  ```

  </details>

- Các thuộc tính chính:
  - `getParameter()`: Lấy các tham số
  - `getSupportedExtensions()`: Lấy các phần mở rộng
  - `getShaderPrecisionFormat()`: Lấy định dạng shader

#### WebGL Extensions

- ID: `webgl_extensions`
- Mô tả: Lấy danh sách các phần mở rộng WebGL được hỗ trợ
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  webglContext.getSupportedExtensions().join(";");
  ```

  </details>

- Ví dụ: `"ANGLE_instanced_arrays;EXT_blend_minmax;EXT_clip_control;EXT_color_buffer_half_float;EXT_depth_clamp;EXT_disjoint_timer_query;EXT_float_blend;EXT_frag_depth;EXT_polygon_offset_clamp;EXT_shader_texture_lod;EXT_texture_compression_bptc;EXT_texture_compression_rgtc;EXT_texture_filter_anisotropic;EXT_texture_mirror_clamp_to_edge;EXT_sRGB;KHR_parallel_shader_compile;OES_element_index_uint;OES_fbo_render_mipmap;OES_standard_derivatives;OES_texture_float;OES_texture_float_linear;OES_texture_half_float;OES_texture_half_float_linear;OES_vertex_array_object;WEBGL_blend_func_extended;WEBGL_color_buffer_float;WEBGL_compressed_texture_s3tc;WEBGL_compressed_texture_s3tc_srgb;WEBGL_debug_renderer_info;WEBGL_debug_shaders;WEBGL_depth_texture;WEBGL_draw_buffers;WEBGL_lose_context;WEBGL_multi_draw;WEBGL_polygon_mode"`

#### WebGL Extensions Hash

- ID: `webgl_extensions_hash`
- Mô tả: Băm danh sách phần mở rộng WebGL bằng MurmurHash3
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  hashFunctions.murmurHash3(webglContext.getSupportedExtensions().join(";"), 0);
  ```

  </details>

- Ví dụ: `"7300c23f4e6fa34e534fc99c1b628588"`

#### WebGL Renderer

- ID: `webgl_renderer`
- Mô tả: Lấy thông tin về trình kết xuất WebGL
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  webglContext.getParameter(webglContext.RENDERER);
  ```

  </details>

- Ví dụ: `"WebKit WebGL"`

#### WebGL Vendor

- ID: `webgl_vendor`
- Mô tả: Lấy thông tin về nhà cung cấp WebGL
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  webglContext.getParameter(webglContext.VENDOR);
  ```

  </details>

- Ví dụ: `"WebKit"`

#### WebGL Version

- ID: `webgl_version`
- Mô tả: Lấy thông tin về phiên bản WebGL
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  webglContext.getParameter(webglContext.VERSION);
  ```

  </details>

- Ví dụ: `"WebGL 1.0 (OpenGL ES 2.0 Chromium)"`

#### WebGL Shading Language Version

- ID: `webgl_shading_language_version`
- Mô tả: Lấy thông tin về phiên bản ngôn ngữ shader WebGL
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  webglContext.getParameter(webglContext.SHADING_LANGUAGE_VERSION);
  ```

  </details>

- Ví dụ: `"WebGL GLSL ES 1.0 (OpenGL ES GLSL ES 1.0 Chromium)"`

#### WebGL Aliased Line Width Range

- ID: `webgl_aliased_line_width_range`
- Mô tả: Lấy phạm vi độ rộng đường được hỗ trợ
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  const formatRange = function (t, e) {
  	return t.clearColor(0, 0, 0, 1), t.enable(t.DEPTH_TEST), t.depthFunc(t.LEQUAL), t.clear(t.COLOR_BUFFER_BIT | t.DEPTH_BUFFER_BIT), `[${e[0]}, ${e[1]}]`;
  };
  formatRange(webglContext, webglContext.getParameter(webglContext.ALIASED_LINE_WIDTH_RANGE));
  ```

  </details>

- Ví dụ: `"[1, 1]"`

#### WebGL Aliased Point Size Range

- ID: `webgl_aliased_point_size_range`
- Mô tả: Lấy phạm vi kích thước điểm được hỗ trợ
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  const formatRange = function (t, e) {
  	return t.clearColor(0, 0, 0, 1), t.enable(t.DEPTH_TEST), t.depthFunc(t.LEQUAL), t.clear(t.COLOR_BUFFER_BIT | t.DEPTH_BUFFER_BIT), `[${e[0]}, ${e[1]}]`;
  };
  formatRange(webglContext, webglContext.getParameter(webglContext.ALIASED_POINT_SIZE_RANGE));
  ```

  </details>

- Ví dụ: `"[1, 1024]"`

#### WebGL Antialiasing

- ID: `webgl_antialiasing`
- Mô tả: Kiểm tra xem khử răng cưa có được bật không
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  webglContext.getContextAttributes()?.antialias ? "yes" : "no";
  ```

  </details>

- Ví dụ: `"yes"`

#### WebGL Bits

- ID: `webgl_bits`
- Mô tả: Lấy thông tin về số bit của các kênh màu và độ sâu
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  [webglContext.getParameter(webglContext.ALPHA_BITS), webglContext.getParameter(webglContext.BLUE_BITS), webglContext.getParameter(webglContext.DEPTH_BITS), webglContext.getParameter(webglContext.GREEN_BITS), webglContext.getParameter(webglContext.RED_BITS), webglContext.getParameter(webglContext.STENCIL_BITS)].join(",");
  ```

  </details>

- Ví dụ: `"8,8,24,8,8,0"`

#### WebGL Max Params

- ID: `webgl_max_params`
- Mô tả: Lấy các giới hạn tối đa của WebGL
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  [webglContext.getExtension("EXT_texture_filter_anisotropic")?.MAX_TEXTURE_MAX_ANISOTROPY_EXT || 2, webglContext.getParameter(webglContext.MAX_COMBINED_TEXTURE_IMAGE_UNITS), webglContext.getParameter(webglContext.MAX_CUBE_MAP_TEXTURE_SIZE), webglContext.getParameter(webglContext.MAX_FRAGMENT_UNIFORM_VECTORS), webglContext.getParameter(webglContext.MAX_RENDERBUFFER_SIZE), webglContext.getParameter(webglContext.MAX_TEXTURE_IMAGE_UNITS), webglContext.getParameter(webglContext.MAX_TEXTURE_SIZE), webglContext.getParameter(webglContext.MAX_VARYING_VECTORS), webglContext.getParameter(webglContext.MAX_VERTEX_ATTRIBS), webglContext.getParameter(webglContext.MAX_VERTEX_TEXTURE_IMAGE_UNITS), webglContext.getParameter(webglContext.MAX_VERTEX_UNIFORM_VECTORS)].join(",");
  ```

  </details>

- Ví dụ: `"16,32,16384,1024,16384,16,16384,30,16,16,4095"`

#### WebGL Max Viewport Dimensions

- ID: `webgl_max_viewport_dims`
- Mô tả: Lấy kích thước viewport tối đa được hỗ trợ
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  const formatRange = function (t, e) {
  	return t.clearColor(0, 0, 0, 1), t.enable(t.DEPTH_TEST), t.depthFunc(t.LEQUAL), t.clear(t.COLOR_BUFFER_BIT | t.DEPTH_BUFFER_BIT), `[${e[0]}, ${e[1]}]`;
  };
  formatRange(webglContext, webglContext.getParameter(webglContext.MAX_VIEWPORT_DIMS));
  ```

  </details>

- Ví dụ: `"[32767, 32767]"`

#### WebGL Unmasked Vendor

- ID: `webgl_unmasked_vendor`
- Mô tả: Lấy thông tin thực về nhà cung cấp WebGL
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  const debugInfo = webglContext.getExtension("WEBGL_debug_renderer_info");
  webglContext.getParameter(debugInfo?.UNMASKED_VENDOR_WEBGL);
  ```

  </details>

- Ví dụ: `"Google Inc. (NVIDIA)"`

#### WebGL Unmasked Renderer

- ID: `webgl_unmasked_renderer`
- Mô tả: Lấy thông tin thực về trình kết xuất WebGL
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  const debugInfo = webglContext.getExtension("WEBGL_debug_renderer_info");
  webglContext.getParameter(debugInfo?.UNMASKED_RENDERER_WEBGL);
  ```

  </details>

- Ví dụ: `"ANGLE (NVIDIA, NVIDIA GeForce RTX 4070 Ti SUPER (0x00002705) Direct3D11 vs_5_0 ps_5_0, D3D11)"`

#### WebGL Vertex Shader Float Parameters

- ID: `webgl_vsf_params`
- Mô tả: Lấy thông tin về độ chính xác số thực của vertex shader
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  [webglContext.getShaderPrecisionFormat(webglContext.VERTEX_SHADER, webglContext.HIGH_FLOAT)?.precision, webglContext.getShaderPrecisionFormat(webglContext.VERTEX_SHADER, webglContext.HIGH_FLOAT)?.rangeMin, webglContext.getShaderPrecisionFormat(webglContext.VERTEX_SHADER, webglContext.HIGH_FLOAT)?.rangeMax, webglContext.getShaderPrecisionFormat(webglContext.VERTEX_SHADER, webglContext.MEDIUM_FLOAT)?.precision, webglContext.getShaderPrecisionFormat(webglContext.VERTEX_SHADER, webglContext.MEDIUM_FLOAT)?.rangeMin, webglContext.getShaderPrecisionFormat(webglContext.VERTEX_SHADER, webglContext.MEDIUM_FLOAT)?.rangeMax, webglContext.getShaderPrecisionFormat(webglContext.VERTEX_SHADER, webglContext.LOW_FLOAT)?.precision, webglContext.getShaderPrecisionFormat(webglContext.VERTEX_SHADER, webglContext.LOW_FLOAT)?.rangeMin, webglContext.getShaderPrecisionFormat(webglContext.VERTEX_SHADER, webglContext.LOW_FLOAT)?.rangeMax].join(",");
  ```

  </details>

- Ví dụ: `"23,127,127,23,127,127,23,127,127"`

#### WebGL Vertex Shader Integer Parameters

- ID: `webgl_vsi_params`
- Mô tả: Lấy thông tin về độ chính xác số nguyên của vertex shader
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  [webglContext.getShaderPrecisionFormat(webglContext.VERTEX_SHADER, webglContext.HIGH_INT)?.precision, webglContext.getShaderPrecisionFormat(webglContext.VERTEX_SHADER, webglContext.HIGH_INT)?.rangeMin, webglContext.getShaderPrecisionFormat(webglContext.VERTEX_SHADER, webglContext.HIGH_INT)?.rangeMax, webglContext.getShaderPrecisionFormat(webglContext.VERTEX_SHADER, webglContext.MEDIUM_INT)?.precision, webglContext.getShaderPrecisionFormat(webglContext.VERTEX_SHADER, webglContext.MEDIUM_INT)?.rangeMin, webglContext.getShaderPrecisionFormat(webglContext.VERTEX_SHADER, webglContext.MEDIUM_INT)?.rangeMax, webglContext.getShaderPrecisionFormat(webglContext.VERTEX_SHADER, webglContext.LOW_INT)?.precision, webglContext.getShaderPrecisionFormat(webglContext.VERTEX_SHADER, webglContext.LOW_INT)?.rangeMin, webglContext.getShaderPrecisionFormat(webglContext.VERTEX_SHADER, webglContext.LOW_INT)?.rangeMax].join(",");
  ```

  </details>

- Ví dụ: `"0,31,30,0,31,30,0,31,30"`

#### WebGL Fragment Shader Float Parameters

- ID: `webgl_fsf_params`
- Mô tả: Lấy thông tin về độ chính xác số thực của fragment shader
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  [webglContext.getShaderPrecisionFormat(webglContext.FRAGMENT_SHADER, webglContext.HIGH_FLOAT)?.precision, webglContext.getShaderPrecisionFormat(webglContext.FRAGMENT_SHADER, webglContext.HIGH_FLOAT)?.rangeMin, webglContext.getShaderPrecisionFormat(webglContext.FRAGMENT_SHADER, webglContext.HIGH_FLOAT)?.rangeMax, webglContext.getShaderPrecisionFormat(webglContext.FRAGMENT_SHADER, webglContext.MEDIUM_FLOAT)?.precision, webglContext.getShaderPrecisionFormat(webglContext.FRAGMENT_SHADER, webglContext.MEDIUM_FLOAT)?.rangeMin, webglContext.getShaderPrecisionFormat(webglContext.FRAGMENT_SHADER, webglContext.MEDIUM_FLOAT)?.rangeMax, webglContext.getShaderPrecisionFormat(webglContext.FRAGMENT_SHADER, webglContext.LOW_FLOAT)?.precision, webglContext.getShaderPrecisionFormat(webglContext.FRAGMENT_SHADER, webglContext.LOW_FLOAT)?.rangeMin, webglContext.getShaderPrecisionFormat(webglContext.FRAGMENT_SHADER, webglContext.LOW_FLOAT)?.rangeMax].join(",");
  ```

  </details>

- Ví dụ: `"23,127,127,23,127,127,23,127,127"`

#### WebGL Fragment Shader Integer Parameters

- ID: `webgl_fsi_params`
- Mô tả: Lấy thông tin về độ chính xác số nguyên của fragment shader
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  [webglContext.getShaderPrecisionFormat(webglContext.FRAGMENT_SHADER, webglContext.HIGH_INT)?.precision, webglContext.getShaderPrecisionFormat(webglContext.FRAGMENT_SHADER, webglContext.HIGH_INT)?.rangeMin, webglContext.getShaderPrecisionFormat(webglContext.FRAGMENT_SHADER, webglContext.HIGH_INT)?.rangeMax, webglContext.getShaderPrecisionFormat(webglContext.FRAGMENT_SHADER, webglContext.MEDIUM_INT)?.precision, webglContext.getShaderPrecisionFormat(webglContext.FRAGMENT_SHADER, webglContext.MEDIUM_INT)?.rangeMin, webglContext.getShaderPrecisionFormat(webglContext.FRAGMENT_SHADER, webglContext.MEDIUM_INT)?.rangeMax, webglContext.getShaderPrecisionFormat(webglContext.FRAGMENT_SHADER, webglContext.LOW_INT)?.precision, webglContext.getShaderPrecisionFormat(webglContext.FRAGMENT_SHADER, webglContext.LOW_INT)?.rangeMin, webglContext.getShaderPrecisionFormat(webglContext.FRAGMENT_SHADER, webglContext.LOW_INT)?.rangeMax].join(",");
  ```

  </details>

- Ví dụ: `"0,31,30,0,31,30,0,31,30"`

#### WebGL Parameters Hash

- ID: `webgl_hash_webgl`
- Mô tả: Băm tất cả các tham số WebGL bằng MurmurHash3
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  const webglParams = {
  	webgl_extensions: "ANGLE_instanced_arrays;EXT_blend_minmax;EXT_clip_control;EXT_color_buffer_half_float;EXT_depth_clamp;EXT_disjoint_timer_query;EXT_float_blend;EXT_frag_depth;EXT_polygon_offset_clamp;EXT_shader_texture_lod;EXT_texture_compression_bptc;EXT_texture_compression_rgtc;EXT_texture_filter_anisotropic;EXT_texture_mirror_clamp_to_edge;EXT_sRGB;KHR_parallel_shader_compile;OES_element_index_uint;OES_fbo_render_mipmap;OES_standard_derivatives;OES_texture_float;OES_texture_float_linear;OES_texture_half_float;OES_texture_half_float_linear;OES_vertex_array_object;WEBGL_blend_func_extended;WEBGL_color_buffer_float;WEBGL_compressed_texture_s3tc;WEBGL_compressed_texture_s3tc_srgb;WEBGL_debug_renderer_info;WEBGL_debug_shaders;WEBGL_depth_texture;WEBGL_draw_buffers;WEBGL_lose_context;WEBGL_multi_draw;WEBGL_polygon_mode",
  	webgl_extensions_hash: "7300c23f4e6fa34e534fc99c1b628588",
  	webgl_renderer: "WebKit WebGL",
  	webgl_vendor: "WebKit",
  	webgl_version: "WebGL 1.0 (OpenGL ES 2.0 Chromium)",
  	webgl_shading_language_version: "WebGL GLSL ES 1.0 (OpenGL ES GLSL ES 1.0 Chromium)",
  	webgl_aliased_line_width_range: "[1, 1]",
  	webgl_aliased_point_size_range: "[1, 1024]",
  	webgl_antialiasing: "yes",
  	webgl_bits: "8,8,24,8,8,0",
  	webgl_max_params: "16,32,16384,1024,16384,16,16384,30,16,16,4095",
  	webgl_max_viewport_dims: "[32767, 32767]",
  	webgl_unmasked_vendor: "Google Inc. (NVIDIA)",
  	webgl_unmasked_renderer: "ANGLE (NVIDIA, NVIDIA GeForce RTX 4070 Ti SUPER (0x00002705) Direct3D11 vs_5_0 ps_5_0, D3D11)",
  	webgl_vsf_params: "23,127,127,23,127,127,23,127,127",
  	webgl_vsi_params: "0,31,30,0,31,30,0,31,30",
  	webgl_fsf_params: "23,127,127,23,127,127,23,127,127",
  	webgl_fsi_params: "0,31,30,0,31,30,0,31,30",
  	webgl_hash_webgl: "",
  };

  hashFunctions.murmurHash3(
  	Object.entries(webglParams)
  		.map(([key, value]) => `${key},${value}`)
  		.join(","),
  	0
  );
  ```

  </details>

- **Lưu ý**: Chuỗi sau được nối và băm:

  ```text
  webgl_extensions,ANGLE_instanced_arrays;EXT_blend_minmax;EXT_clip_control;EXT_color_buffer_half_float;EXT_depth_clamp;EXT_disjoint_timer_query;EXT_float_blend;EXT_frag_depth;EXT_polygon_offset_clamp;EXT_shader_texture_lod;EXT_texture_compression_bptc;EXT_texture_compression_rgtc;EXT_texture_filter_anisotropic;EXT_texture_mirror_clamp_to_edge;EXT_sRGB;KHR_parallel_shader_compile;OES_element_index_uint;OES_fbo_render_mipmap;OES_standard_derivatives;OES_texture_float;OES_texture_float_linear;OES_texture_half_float;OES_texture_half_float_linear;OES_vertex_array_object;WEBGL_blend_func_extended;WEBGL_color_buffer_float;WEBGL_compressed_texture_s3tc;WEBGL_compressed_texture_s3tc_srgb;WEBGL_debug_renderer_info;WEBGL_debug_shaders;WEBGL_depth_texture;WEBGL_draw_buffers;WEBGL_lose_context;WEBGL_multi_draw;WEBGL_polygon_mode,webgl_extensions_hash,7300c23f4e6fa34e534fc99c1b628588,webgl_renderer,WebKit WebGL,webgl_vendor,WebKit,webgl_version,WebGL 1.0 (OpenGL ES 2.0 Chromium),webgl_shading_language_version,WebGL GLSL ES 1.0 (OpenGL ES GLSL ES 1.0 Chromium),webgl_aliased_line_width_range,[1, 1],webgl_aliased_point_size_range,[1, 1024],webgl_antialiasing,yes,webgl_bits,8,8,24,8,8,0,webgl_max_params,16,32,16384,1024,16384,16,16384,30,16,16,4095,webgl_max_viewport_dims,[32767, 32767],webgl_unmasked_vendor,Google Inc. (NVIDIA),webgl_unmasked_renderer,ANGLE (NVIDIA, NVIDIA GeForce RTX 4070 Ti SUPER (0x00002705) Direct3D11 vs_5_0 ps_5_0, D3D11),webgl_vsf_params,23,127,127,23,127,127,23,127,127,webgl_vsi_params,0,31,30,0,31,30,0,31,30,webgl_fsf_params,23,127,127,23,127,127,23,127,127,webgl_fsi_params,0,31,30,0,31,30,0,31,30,webgl_hash_webgl,
  ```

- Có một `webgl_hash_webgl,` với một chuỗi rỗng, điều này đã bị bỏ sót trong một số trình giải quyết và tài liệu công khai trước đây
- Ví dụ: `"80bd944fa021d5a6867dd72a5270b2ba"`

### Enhanced Fingerprint

- ID: `enhanced_fp`
- Childs:
  - [WebGL Rendering Context](#webgl-rendering-context)
- Mô tả: Tập hợp dữ liệu chi tiết về fingerprint trình duyệt tập trung vào khả năng WebGL và thông tin phần cứng

#### User Agent Data Brands

- ID: `user_agent_data_brands`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  navigator.userAgentData && navigator.userAgentData.brands
  	? navigator.userAgentData.brands
  			.map(function (t) {
  				return t.brand;
  			})
  			.join(",")
  	: null;
  ```

  </details>

- Ví dụ: `"Chromium,Not_A Brand"`

#### User Agent Data Mobile

- ID: `user_agent_data_mobile`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  navigator.userAgentData ? (navigator.userAgentData.mobile === undefined ? null : navigator.userAgentData.mobile) : null;
  ```

  </details>

- Ví dụ: `false`

#### Navigator Connection Downlink

- ID: `navigator_connection_downlink`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (navigator.connection && navigator.connection.downlink) || null;
  ```

  </details>

- **Lưu ý**: Giá trị `navigator_connection_downlink` luôn chia hết cho 0.05 (tức là `navigator_connection_downlink % 0.05 === 0`)
- Ví dụ: `1.65`

#### Navigator Connection Downlink Max

- ID: `navigator_connection_downlink_max`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  navigator.connection && navigator.connection.downlinkMax ? (typeof navigator.connection.downlinkMax === "number" && navigator.connection.downlinkMax !== Infinity ? navigator.connection.downlinkMax : -1) : null;
  ```

  </details>

- Ví dụ: `null`

#### Navigator Connection RTT

- ID: `navigator_connection_rtt`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (navigator.connection && navigator.connection.rtt) || null;
  ```

  </details>

- **Lưu ý**: Giá trị `navigator_connection_rtt` luôn chia hết cho 50 (tức là `navigator_connection_rtt % 50 === 0`)
- Ví dụ: `200`

#### Navigator Connection Save Data

- ID: `navigator_connection_save_data`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  navigator.connection ? (navigator.connection.saveData === undefined ? null : navigator.connection.saveData) : null;
  ```

  </details>

- Ví dụ: `false`

#### Navigator Connection Type

- ID: `navigator_connection_type`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (navigator.connection && navigator.connection.type) || null;
  ```

  </details>

- Tất cả giá trị có thể: [`"wifi"`, `"ethernet"`, `"cellular"`, `"none"`, `"unknown"`]
- Ví dụ: `null`

#### Screen Pixel Depth

- ID: `screen_pixel_depth`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  function isNumber(value) {
  	return typeof value === "number" ? value : null;
  }
  isNumber(screen.pixelDepth);
  ```

  </details>

- Ví dụ: `24`

#### Navigator Device Memory

- ID: `navigator_device_memory`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  function isNumber(value) {
  	return typeof value === "number" ? value : null;
  }
  isNumber(navigator.deviceMemory);
  ```

  </details>

- Tất cả giá trị có thể: [`0.5`, `1`, `2`, `4`, `8`]
- **Lưu ý**: Giá trị luôn là bội số của `0.5`
- **Lưu ý**: Giá trị tối đa là `8` ngay cả khi thiết bị có nhiều bộ nhớ hơn.
- Ví dụ: `8`

#### Navigator PDF Viewer Enabled

- ID: `navigator_pdf_viewer_enabled`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  navigator.pdfViewerEnabled === undefined ? null : navigator.pdfViewerEnabled;
  ```

  </details>

- Ví dụ: `true`

#### Navigator Languages

- ID: `navigator_languages`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  navigator.languages && typeof navigator.languages.join === "function" ? navigator.languages.join(",") : null;
  ```

  </details>

- Ví dụ: `"en-GB"`

#### Window Inner Width

- ID: `window_inner_width`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  function isNumber(value) {
  	return typeof value === "number" ? value : null;
  }
  isNumber(window.innerWidth);
  ```

  </details>

- Ví dụ: `0`

#### Window Inner Height

- ID: `window_inner_height`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  function isNumber(value) {
  	return typeof value === "number" ? value : null;
  }
  isNumber(window.innerHeight);
  ```

  </details>

- Ví dụ: `0`

#### Window Outer Width

- ID: `window_outer_width`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  function isNumber(value) {
  	return typeof value === "number" ? value : null;
  }
  isNumber(window.outerWidth);
  ```

  </details>

- Ví dụ: `1265`

#### Window Outer Height

- ID: `window_outer_height`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  function isNumber(value) {
  	return typeof value === "number" ? value : null;
  }
  isNumber(window.outerHeight);
  ```

  </details>

- Ví dụ: `1372`

#### Browser Detection Firefox

- ID: `browser_detection_firefox`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  navigator.userAgent ? navigator.userAgent.indexOf("Firefox") > 0 : null;
  ```

  </details>

- Ví dụ: `false`

#### Browser Detection Brave

- ID: `browser_detection_brave`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  !!navigator.brave;
  ```

  </details>

- Ví dụ: `false`

#### Browser API Checks

- ID: `browser_api_checks`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	try {
  		return ["permission_status: " + (!!window.PermissionStatus && Object.prototype.hasOwnProperty.call(window.PermissionStatus.prototype, "name")), "eye_dropper: " + !!window.EyeDropper, "audio_data: " + !!window.AudioData, "writable_stream: " + !!window.WritableStreamDefaultController, "css_style_rule: " + !!window.CSSCounterStyleRule, "navigator_ua: " + !!window.NavigatorUAData, "barcode_detector: " + !!window.BarcodeDetector, "display_names: " + !(!window.Intl || !window.Intl.DisplayNames), "contacts_manager: " + !!(navigator && navigator.contacts && navigator.ContactsManager), "svg_discard_element: " + !!window.SVGDiscardElement, "usb: " + (navigator.usb ? "defined" : "NA"), "media_device: " + (navigator.mediaDevices ? "defined" : "NA"), "playback_quality: " + !!(window.HTMLVideoElement && window.HTMLVideoElement.prototype && window.HTMLVideoElement.prototype.getVideoPlaybackQuality)];
  	} catch (t) {
  		return null;
  	}
  })();
  ```

  </details>

- Ví dụ:

  ```javascript
  ["permission_status: true", "eye_dropper: true", "audio_data: true", "writable_stream: true", "css_style_rule: true", "navigator_ua: true", "barcode_detector: false", "display_names: true", "contacts_manager: false", "svg_discard_element: false", "usb: defined", "media_device: defined", "playback_quality: true"];
  ```

#### Browser Object Checks

- ID: `browser_object_checks`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	const browserObjects = ["chrome", "safari", "__crWeb", "__gCrWeb", "yandex", "__yb", "__ybro", "__firefox__", "firefox", "__edgeTrackingPreventionStatistics", "webkit", "oprt", "samsungAr", "ucweb", "UCShellJava", "puffinDevice", "opr"].reduce(function (detected, name) {
  		if (window[name] && Object.prototype.toString.call(window[name]) === "[object Object]") {
  			return [...detected, name];
  		}
  		return detected;
  	}, []);

  	if (browserObjects.length > 0) {
  		// Giá trị có thể: ["chrome"] nếu bạn đang sử dụng Chrome
  		return hashFunctions.md5(browserObjects.sort().join(","));
  	}
  })();
  ```

  </details>

- Ví dụ: `"554838a8451ac36cb977e719e9d6623c"`

#### Sandbox Detection

- ID: `29s83ih9`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	try {
  		var isProcess = typeof process !== "undefined",
  			isGlobal = typeof n.g !== "undefined" && {}.toString.call(n.g) === "[object global]",
  			isSetImmediate = typeof setImmediate !== "undefined" && typeof window === "undefined",
  			isModule = false;
  		window.module !== t && (t.path || t.filename || t.paths) && (isModule = true);
  		var isVirtualConsole = false;
  		"_virtualConsole" in window && (isVirtualConsole = true);
  		var result = isProcess || isModule || isGlobal || isSetImmediate || isVirtualConsole;
  		// Giá trị có thể: ["true", "false"]
  		return "".concat(hashFunctions.md5(result.toString())).concat(result ? "\u2062" : "\u2063");
  	} catch (t) {
  		// Giá trị có thể: ["FAIL"]
  		return "".concat(hashFunctions.md5("FAIL"), "\u2064");
  	}
  })();
  ```

  </details>

- Ví dụ: `"68934a3e9455fa72420237eb05902327⁣"`

#### Audio Codecs

- ID: `audio_codecs`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	const audio = document.createElement("audio");
  	let codecs = null;
  	if (audio.canPlayType) {
  		codecs = {
  			ogg: audio.canPlayType('audio/ogg; codecs="vorbis"'),
  			mp3: audio.canPlayType("audio/mpeg;"),
  			wav: audio.canPlayType('audio/wav; codecs="1"'),
  			m4a: audio.canPlayType("audio/x-m4a;"),
  			aac: audio.canPlayType("audio/aac;"),
  		};
  	}
  	return JSON.stringify(codecs);
  })();
  ```

  </details>

- Ví dụ:
  - JSON gốc: `{"ogg":"probably","mp3":"probably","wav":"probably","m4a":"","aac":""}`
  - Escaped cho BDA: `"{\"ogg\":\"probably\",\"mp3\":\"probably\",\"wav\":\"probably\",\"m4a\":\"\",\"aac\":\"\"}"`

#### Audio Codecs Extended Hash

- ID: `audio_codecs_extended_hash`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  hashFunctions.md5(
  	(function () {
  		const audioCodecs = {};
  		const audioElement = document.createElement("audio");
  		[
  			'audio/mp4; codecs="mp4a.40"',
  			'audio/mp4; codecs="mp4a.40.1"',
  			'audio/mp4; codecs="mp4a.40.2"',
  			'audio/mp4; codecs="mp4a.40.3"',
  			'audio/mp4; codecs="mp4a.40.4"',
  			'audio/mp4; codecs="mp4a.40.5"',
  			'audio/mp4; codecs="mp4a.40.6"',
  			'audio/mp4; codecs="mp4a.40.7"',
  			'audio/mp4; codecs="mp4a.40.8"',
  			'audio/mp4; codecs="mp4a.40.9"',
  			'audio/mp4; codecs="mp4a.40.12"',
  			'audio/mp4; codecs="mp4a.40.13"',
  			'audio/mp4; codecs="mp4a.40.14"',
  			'audio/mp4; codecs="mp4a.40.15"',
  			'audio/mp4; codecs="mp4a.40.16"',
  			'audio/mp4; codecs="mp4a.40.17"',
  			'audio/mp4; codecs="mp4a.40.19"',
  			'audio/mp4; codecs="mp4a.40.20"',
  			'audio/mp4; codecs="mp4a.40.21"',
  			'audio/mp4; codecs="mp4a.40.22"',
  			'audio/mp4; codecs="mp4a.40.23"',
  			'audio/mp4; codecs="mp4a.40.24"',
  			'audio/mp4; codecs="mp4a.40.25"',
  			'audio/mp4; codecs="mp4a.40.26"',
  			'audio/mp4; codecs="mp4a.40.27"',
  			'audio/mp4; codecs="mp4a.40.28"',
  			'audio/mp4; codecs="mp4a.40.29"',
  			'audio/mp4; codecs="mp4a.40.32"',
  			'audio/mp4; codecs="mp4a.40.33"',
  			'audio/mp4; codecs="mp4a.40.34"',
  			'audio/mp4; codecs="mp4a.40.35"',
  			'audio/mp4; codecs="mp4a.40.36"',
  			'audio/mp4; codecs="mp4a.66"',
  			'audio/mp4; codecs="mp4a.67"',
  			'audio/mp4; codecs="mp4a.68"',
  			'audio/mp4; codecs="mp4a.69"',
  			'audio/mp4; codecs="mp4a.6B"',
  			'audio/mp4; codecs="mp3"',
  			'audio/mp4; codecs="flac"',
  			'audio/mp4; codecs="bogus"',
  			'audio/mp4; codecs="aac"',
  			'audio/mp4; codecs="ac3"',
  			'audio/mp4; codecs="A52"',
  			'audio/mpeg; codecs="mp3"',
  			'audio/wav; codecs="0"',
  			'audio/wav; codecs="2"',
  			'audio/wave; codecs="0"',
  			'audio/wave; codecs="1"',
  			'audio/wave; codecs="2"',
  			'audio/x-wav; codecs="0"',
  			'audio/x-wav; codecs="1"',
  			'audio/x-wav; codecs="2"',
  			'audio/x-pn-wav; codecs="0"',
  			'audio/x-pn-wav; codecs="1"',
  			'audio/x-pn-wav; codecs="2"',
  		].forEach(function (codec) {
  			let canPlayResult = null;
  			if (audioElement.canPlayType) {
  				canPlayResult = audioElement.canPlayType(codec);
  			}
  			let mediaSourceResult = null;
  			if (window.MediaSource && window.MediaSource.isTypeSupported) {
  				mediaSourceResult = window.MediaSource.isTypeSupported(codec);
  			}
  			audioCodecs[codec] = {
  				canPlay: canPlayResult,
  				mediaSource: mediaSourceResult,
  			};
  		});
  		return JSON.stringify(audioCodecs);
  	})()
  );
  ```

  </details>

- JSON gốc: `"{\"audio/mp4; codecs=\\\"mp4a.40\\\"\":{\"canPlay\":\"maybe\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.1\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.2\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":true},\"audio/mp4; codecs=\\\"mp4a.40.3\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.4\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.5\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":true},\"audio/mp4; codecs=\\\"mp4a.40.6\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.7\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.8\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.9\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.12\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.13\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.14\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.15\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.16\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.17\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.19\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.20\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.21\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.22\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.23\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.24\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.25\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.26\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.27\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.28\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.29\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":true},\"audio/mp4; codecs=\\\"mp4a.40.32\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.33\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.34\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.35\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.40.36\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.66\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.67\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":true},\"audio/mp4; codecs=\\\"mp4a.68\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.69\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp4a.6B\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"mp3\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"flac\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":true},\"audio/mp4; codecs=\\\"bogus\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"aac\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"ac3\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mp4; codecs=\\\"A52\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/mpeg; codecs=\\\"mp3\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":false},\"audio/wav; codecs=\\\"0\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/wav; codecs=\\\"2\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/wave; codecs=\\\"0\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/wave; codecs=\\\"1\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/wave; codecs=\\\"2\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/x-wav; codecs=\\\"0\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/x-wav; codecs=\\\"1\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":false},\"audio/x-wav; codecs=\\\"2\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/x-pn-wav; codecs=\\\"0\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/x-pn-wav; codecs=\\\"1\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"audio/x-pn-wav; codecs=\\\"2\\\"\":{\"canPlay\":\"\",\"mediaSource\":false}}"`
- Ví dụ: `"2cb10f57f3e66b66beab51c9d6ab0e24"`

#### Video Codecs

- ID: `video_codecs`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	const videoElement = document.createElement("video");
  	let codecSupport = null;
  	if (videoElement.canPlayType) {
  		codecSupport = {
  			ogg: videoElement.canPlayType('video/ogg; codecs="theora"'),
  			h264: videoElement.canPlayType('video/mp4; codecs="avc1.42E01E"'),
  			webm: videoElement.canPlayType('video/webm; codecs="vp8, vorbis"'),
  			mpeg4v: videoElement.canPlayType('video/mp4; codecs="mp4v.20.8, mp4a.40.2"'),
  			mpeg4a: videoElement.canPlayType('video/mp4; codecs="mp4v.20.240, mp4a.40.2"'),
  			theora: videoElement.canPlayType('video/x-matroska; codecs="theora, vorbis"'),
  		};
  	}
  	return JSON.stringify(codecSupport);
  })();
  ```

  </details>

- Ví dụ:
  - JSON gốc: `"{"ogg":"","h264":"","webm":"probably","mpeg4v":"","mpeg4a":"","theora":""}"`
  - Escaped cho BDA: `"{\"ogg\":\"\",\"h264\":\"\",\"webm\":\"probably\",\"mpeg4v\":\"\",\"mpeg4a\":\"\",\"theora\":\"\"}"`

#### Video Codecs Extended Hash

- ID: `video_codecs_extended_hash`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  hashFunctions.md5(
  	(function () {
  		const videoCodecs = {};
  		const videoElement = document.createElement("video");
  		[
  			'video/mp4; codecs="hev1.1.6.L93.90"',
  			'video/mp4; codecs="hvc1.1.6.L93.90"',
  			'video/mp4; codecs="hev1.1.6.L93.B0"',
  			'video/mp4; codecs="hvc1.1.6.L93.B0"',
  			'video/mp4; codecs="vp09.00.10.08"',
  			'video/mp4; codecs="vp09.00.50.08"',
  			'video/mp4; codecs="vp09.01.20.08.01"',
  			'video/mp4; codecs="vp09.01.20.08.01.01.01.01.00"',
  			'video/mp4; codecs="vp09.02.10.10.01.09.16.09.01"',
  			'video/mp4; codecs="av01.0.08M.08"',
  			'video/webm; codecs="vorbis"',
  			'video/webm; codecs="vp8"',
  			'video/webm; codecs="vp8.0"',
  			'video/webm; codecs="vp8.0, vorbis"',
  			'video/webm; codecs="vp8, opus"',
  			'video/webm; codecs="vp9"',
  			'video/webm; codecs="vp9, vorbis"',
  			'video/webm; codecs="vp9, opus"',
  			'video/x-matroska; codecs="theora"',
  			'application/x-mpegURL; codecs="avc1.42E01E"',
  			'video/ogg; codecs="dirac, vorbis"',
  			'video/ogg; codecs="theora, speex"',
  			'video/ogg; codecs="theora, vorbis"',
  			'video/ogg; codecs="theora, flac"',
  			'video/ogg; codecs="dirac, flac"',
  			'video/ogg; codecs="flac"',
  			'video/3gpp; codecs="mp4v.20.8, samr"',
  		].forEach(function (codec) {
  			let canPlayResult = null;
  			if (videoElement.canPlayType) {
  				canPlayResult = videoElement.canPlayType(codec);
  			}
  			let mediaSourceResult = null;
  			if (window.MediaSource && window.MediaSource.isTypeSupported) {
  				mediaSourceResult = window.MediaSource.isTypeSupported(codec);
  			}
  			videoCodecs[codec] = {
  				canPlay: canPlayResult,
  				mediaSource: mediaSourceResult,
  			};
  		});
  		return JSON.stringify(videoCodecs);
  	})()
  );
  ```

  </details>

- JSON gốc: `"{\"video/mp4; codecs=\\\"hev1.1.6.L93.90\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"video/mp4; codecs=\\\"hvc1.1.6.L93.90\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"video/mp4; codecs=\\\"hev1.1.6.L93.B0\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"video/mp4; codecs=\\\"hvc1.1.6.L93.B0\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"video/mp4; codecs=\\\"vp09.00.10.08\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":true},\"video/mp4; codecs=\\\"vp09.00.50.08\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":true},\"video/mp4; codecs=\\\"vp09.01.20.08.01\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":true},\"video/mp4; codecs=\\\"vp09.01.20.08.01.01.01.01.00\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":true},\"video/mp4; codecs=\\\"vp09.02.10.10.01.09.16.09.01\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":true},\"video/mp4; codecs=\\\"av01.0.08M.08\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":true},\"video/webm; codecs=\\\"vorbis\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":true},\"video/webm; codecs=\\\"vp8\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":true},\"video/webm; codecs=\\\"vp8.0\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":false},\"video/webm; codecs=\\\"vp8.0, vorbis\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":false},\"video/webm; codecs=\\\"vp8, opus\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":true},\"video/webm; codecs=\\\"vp9\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":true},\"video/webm; codecs=\\\"vp9, vorbis\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":true},\"video/webm; codecs=\\\"vp9, opus\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":true},\"video/x-matroska; codecs=\\\"theora\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"application/x-mpegURL; codecs=\\\"avc1.42E01E\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"video/ogg; codecs=\\\"dirac, vorbis\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"video/ogg; codecs=\\\"theora, speex\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"video/ogg; codecs=\\\"theora, vorbis\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"video/ogg; codecs=\\\"theora, flac\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"video/ogg; codecs=\\\"dirac, flac\\\"\":{\"canPlay\":\"\",\"mediaSource\":false},\"video/ogg; codecs=\\\"flac\\\"\":{\"canPlay\":\"probably\",\"mediaSource\":false},\"video/3gpp; codecs=\\\"mp4v.20.8, samr\\\"\":{\"canPlay\":\"\",\"mediaSource\":false}}"`
- Ví dụ: `"cb2c967d0cd625019556b39c63f7d435"`

#### Media Query Dark Mode

- ID: `media_query_dark_mode`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	function matchMediaQuery(feature, values) {
  		if (typeof matchMedia === "undefined") {
  			return "unsupported";
  		}
  		for (const value of values) {
  			const query = matchMedia(`(${feature}:${value})`);
  			if (query.matches || query.msMatchesSelector) {
  				return value;
  			}
  		}
  		return "unknown";
  	}
  	return matchMediaQuery("prefers-color-scheme", ["light", "dark"]) === "dark";
  })();
  ```

  </details>

- Ví dụ: `true`

#### Media Query Count

- ID: `css_media_queries`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	function matchMediaQuery(feature, values) {
  		if (typeof matchMedia === "undefined") {
  			return "unsupported";
  		}
  		for (const value of values) {
  			const query = matchMedia(`(${feature}:${value})`);
  			if (query.matches || query.msMatchesSelector) {
  				return value;
  			}
  		}
  		return "unknown";
  	}

  	const queries = [
  		{
  			attribute: "forced-colors",
  			options: ["none", "active"],
  			bias: "active",
  		},
  		{
  			attribute: "inverted-colors",
  			options: ["inverted", "none"],
  			bias: "inverted",
  		},
  		{
  			attribute: "dynamic-range",
  			options: ["high", "standard"],
  			bias: "high",
  		},
  		{
  			attribute: "prefers-reduced-motion",
  			options: ["reduce", "no-preference"],
  			bias: "reduce",
  		},
  	];

  	return queries.reduce(function (total, query) {
  		const matches = matchMediaQuery(query.attribute, query.options) === query.bias;
  		return total + Number(matches);
  	}, 0);
  })();
  ```

  </details>

- Ví dụ: `1`

#### Color Gamut

- ID: `css_color_gamut`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	function matchMediaQuery(feature, values) {
  		if (typeof matchMedia === "undefined") {
  			return "unsupported";
  		}
  		for (const value of values) {
  			const query = matchMedia(`(${feature}:${value})`);
  			if (query.matches || query.msMatchesSelector) {
  				return value;
  			}
  		}
  		return "unknown";
  	}
  	return matchMediaQuery("color-gamut", ["rec2020", "p3", "srgb"]);
  })();
  ```

  </details>

- Ví dụ: `"srgb"`

#### Contrast Mode

- ID: `css_contrast`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	function matchMediaQuery(feature, values) {
  		if (typeof matchMedia === "undefined") {
  			return "unsupported";
  		}
  		for (const value of values) {
  			const query = matchMedia(`(${feature}:${value})`);
  			if (query.matches || query.msMatchesSelector) {
  				return value;
  			}
  		}
  		return "unknown";
  	}
  	return matchMediaQuery("prefers-contrast", ["low", "less", "no-preference", "more", "high", "forced"]);
  })();
  ```

  </details>

- Ví dụ: `"no-preference"`

#### Monochrome Mode

- ID: `css_monochrome`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	function matchMediaQuery(feature, values) {
  		if (typeof matchMedia === "undefined") {
  			return "unsupported";
  		}
  		for (const value of values) {
  			const query = matchMedia(`(${feature}:${value})`);
  			if (query.matches || query.msMatchesSelector) {
  				return value;
  			}
  		}
  		return "unknown";
  	}
  	function getMonochromeLevel() {
  		const values = Array.from({ length: 11 }, (_, i) => (i * 10).toString());
  		const minMonochrome = matchMediaQuery("min-monochrome", ["0"]);
  		if (minMonochrome === "unknown" || minMonochrome === "unsupported") {
  			return minMonochrome;
  		}
  		return matchMediaQuery("max-monochrome", values);
  	}
  	return "0" !== getMonochromeLevel();
  })();
  ```

  </details>

- Ví dụ: `false`

#### Pointer Type

- ID: `css_pointer`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	function matchMediaQuery(feature, values) {
  		if (typeof matchMedia === "undefined") {
  			return "unsupported";
  		}
  		for (const value of values) {
  			const query = matchMedia(`(${feature}:${value})`);
  			if (query.matches || query.msMatchesSelector) {
  				return value;
  			}
  		}
  		return "unknown";
  	}
  	return matchMediaQuery("any-pointer", ["coarse", "none", "fine"]);
  })();
  ```

  </details>

- Ví dụ: `"fine"`

#### CSS Grid Support

- ID: `css_grid_support`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	function matchMediaQuery(feature, values) {
  		if (typeof matchMedia === "undefined") {
  			return "unsupported";
  		}
  		for (const value of values) {
  			const query = matchMedia(`(${feature}:${value})`);
  			if (query.matches || query.msMatchesSelector) {
  				return value;
  			}
  		}
  		return "unknown";
  	}
  	return "1" === matchMediaQuery("grid", ["0", "1"]);
  })();
  ```

  </details>

- Ví dụ: `false`

#### Headless Browser Phantom

- ID: `headless_browser_phantom`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	const checks = ["callPhantom" in window, "_phantom" in window, "phantom" in window, "_phantomas" in window, "calledPhantom" in window];
  	return checks.some(function (check) {
  		return check === true;
  	});
  })();
  ```

  </details>

- Ví dụ: `false`

#### Headless Browser Selenium

- ID: `headless_browser_selenium`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	try {
  		const webdriverProperties = ["__webdriver_evaluate", "__selenium_evaluate", "__webdriver_script_function", "__webdriver_script_func", "__webdriver_script_fn", "__fxdriver_evaluate", "__driver_unwrapped", "__webdriver_unwrapped", "__driver_evaluate", "__selenium_unwrapped", "__fxdriver_unwrapped"];
  		const seleniumProperties = ["_selenium", "callSelenium", "_Selenium_IDE_Recorder", "webdriver", "calledSelenium"];
  		for (const prop of seleniumProperties) {
  			if (window[prop]) return true;
  		}
  		for (const prop of webdriverProperties) {
  			if (window.document[prop]) return true;
  		}
  		for (const prop in window.document) {
  			if (prop.match(/\$[a-z]dc_/) && window.document[prop].cache_) {
  				return true;
  			}
  		}
  		return !!(window.document.documentElement.getAttribute("selenium") || window.document.documentElement.getAttribute("webdriver") || window.document.documentElement.getAttribute("driver") || navigator.webdriver);
  	} catch (error) {
  		return null;
  	}
  })();
  ```

  </details>

- Ví dụ: `false`

#### Headless Browser Nightmare.js

- ID: `headless_browser_nightmare_js`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	return initData ? initData.nm : null;
  })();
  ```

  </details>

- Ví dụ: `false`

#### Headless Browser Generic

- ID: `headless_browser_generic`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	const documentChecks = [
  		function (t) {
  			return t in window.document;
  		},
  	];
  	const windowChecks = [
  		function (t) {
  			return t in window;
  		},
  	];
  	const documentProps = ["$cdc_asdjflasutopfhvcZLmcfl", "$chrome_asyncScriptInfo", "hidden"];
  	const windowProps = ["cdc_adoQpoasnfa76pfcZLmcfl_Array", "cdc_adoQpoasnfa76pfcZLmcfl_Promise", "cdc_adoQpoasnfa76pfcZLmcfl_Symbol", "OSMJIF", "__$webdriverAsyncExecutor", "__lastWatirAlert", "__lastWatirConfirm", "__lastWatirPrompt", "__webdriverFuncgeb", "__webdriver__chr", "__webdriver_script_function", "awesomium", "watinExpressionError", "watinExpressionResult", "spynner_additional_js_loaded", "fmget_targets", "geb", "blender"];
  	const additionalChecks = [
  		function () {
  			return "domAutomation" in window || "domAutomationController" in window;
  		},
  		function () {
  			return !!(window.external && window.external.toString && window.external.toString().indexOf("Sequentum") > -1);
  		},
  		function () {
  			return (Object.prototype.toString.call(window.process) === "[object Object]" && "type" in window.process && window.process.type === "renderer") || (typeof process !== "undefined" && Object.prototype.toString.call(process.versions) === "[object Object]" && process.versions.electron) || window.close.toString().indexOf("ELECTRON") > -1;
  		},
  	];
  	const allChecks = [
  		...documentProps.map(function (p) {
  			return documentChecks[0].bind(null, p);
  		}),
  		...windowProps.map(function (p) {
  			return windowChecks[0].bind(null, p);
  		}),
  		...additionalChecks,
  	];
  	let result = 0;
  	for (let i = 0; i < allChecks.length; i++) {
  		if (allChecks[i]()) {
  			result |= 1 << i;
  		}
  	}
  	return result;
  })();
  ```

  </details>

- Ví dụ: `4`

#### Special Timestamp

- ID: `1l2l5234ar2`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	try {
  		let isDebuggerPresent = false;
  		const error = new Error();
  		const descriptor = {
  			configurable: false,
  			enumerable: false,
  			get: function () {
  				isDebuggerPresent = true;
  				return "";
  			},
  		};
  		Object.defineProperty(error, "stack", descriptor);
  		console.debug(error);
  		const marker = isDebuggerPresent ? "\u2062" : "\u2063";
  		return Date.now() + marker;
  	} catch (error) {
  		return null;
  	}
  })();
  ```

  </details>

- Ví dụ: `"1735815090588⁣"`

#### Referrer URL

- ID: `document__referrer`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	function getReferrerWithoutQueryParams(referrer) {
  		if (!referrer && typeof referrer !== "string") {
  			return null;
  		}
  		return referrer.split("?")[0];
  	}
  	return getReferrerWithoutQueryParams(document.referrer);
  })();
  ```

  </details>

- Ví dụ: `"https://iframe.arkoselabs.com/"`

#### Ancestor Origins

- ID: `window__ancestor_origins`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	if (window.location.ancestorOrigins) {
  		const origins = [];
  		const ancestorOrigins = window.location.ancestorOrigins;
  		for (let i = 0; i < ancestorOrigins.length; i++) {
  			origins.push(ancestorOrigins[i]);
  		}
  		return origins;
  	}
  	return null;
  })();
  ```

  </details>

- Ví dụ: [`"https://iframe.arkoselabs.com"`, `"https://signup.live.com"`]

#### Window Tree Index

- ID: `window__tree_index`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	function getWindowTreeIndex(window) {
  		const parent = window.parent;
  		if (window === parent) {
  			return [];
  		}
  		const parentIndex = getWindowTreeIndex(parent);
  		let index = -1;
  		for (let i = 0; i < parent.length; i++) {
  			if (window === parent[i]) {
  				index = i;
  				break;
  			}
  		}
  		parentIndex.push(index);
  		return parentIndex;
  	}
  	return getWindowTreeIndex(window);
  })();
  ```

  </details>

- Ví dụ: [`2`, `0`]

#### Window Tree Structure

- ID: `window__tree_structure`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	let result = "";
  	try {
  		result = JSON.stringify(
  			(function getWindowTree(window) {
  				const children = [];
  				for (let i = 0; i < window.length; i++) {
  					children.push(getWindowTree(window[i]));
  				}
  				return children;
  			})(window.top)
  		);
  	} catch (error) {}
  	return result;
  })();
  ```

  </details>

- Ví dụ: `"[[[]],[],[[]]]"`

#### Current URL

- ID: `window__location_href`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	function getReferrerWithoutQueryParams(referrer) {
  		if (!referrer && typeof referrer !== "string") {
  			return null;
  		}
  		return referrer.split("?")[0];
  	}
  	return window.location && window.location.href ? getReferrerWithoutQueryParams(window.location.href).split("#")[0] : null;
  })();
  ```

  </details>

- Ví dụ: `"https://client-api.arkoselabs.com/v2/2.11.3/enforcement.507409183b9903b911945fa68e24c1d9.html"`

#### Client Config Site Data Location Href

- ID: `client_config__sitedata_location_href`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  initData ? initData.chref : null;
  ```

  </details>

- Ví dụ: `"https://iframe.arkoselabs.com/B7D8911C-5CC8-A9A3-35B0-554ACEE604DA/index.html"`

#### Config Language

- ID: `client_config__language`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  initData ? initData.clang : null;
  ```

  </details>

- Ví dụ: `"en-gb"`

#### Service URL

- ID: `client_config__surl`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  initData.surl ? initData.surl : null;
  ```

  </details>

- Ví dụ: `"https://client-api.arkoselabs.com"`

#### Service URL Hash

- ID: `c8480e29a`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function getServiceUrlHash(serviceUrl) {
  	const urlString = serviceUrl ?? "";
  	return hashFunctions.md5(urlString) + (serviceUrl ? "\u2062" : "\u2063");
  })(initData.surl ? initData.surl : null);
  ```

  </details>

- Ví dụ: `"165ee51d4a3e27bfeee660c40851de9f⁢"`

#### Inline Triggered

- ID: `client_config__triggered_inline`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  !!initData && initData.triggeredInline;
  ```

  </details>

- Ví dụ: `false`

#### Mobile SDK Usage

- ID: `mobile_sdk__is_sdk`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  !!initData && initData.sdk;
  ```

  </details>

- Ví dụ: `false`

#### Audio Fingerprint

- ID: `audio_fingerprint`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	return new Promise(function (resolve) {
  		try {
  			if (!window.OfflineAudioContext) {
  				if (!window.webkitOfflineAudioContext) {
  					return void resolve(null);
  				}
  				window.OfflineAudioContext = window.webkitOfflineAudioContext;
  			}
  			const audioContext = new window.OfflineAudioContext(1, 44100, 44100);
  			const oscillator = audioContext.createOscillator();
  			oscillator.type = "triangle";
  			oscillator.frequency.value = 10000;
  			const compressor = audioContext.createDynamicsCompressor();
  			if (compressor.threshold) compressor.threshold.value = -50;
  			if (compressor.knee) compressor.knee.value = 40;
  			if (compressor.ratio) compressor.ratio.value = 12;
  			if (compressor.attack) compressor.attack.value = 0;
  			if (compressor.release) compressor.release.value = 0.25;
  			oscillator.connect(compressor);
  			compressor.connect(audioContext.destination);
  			oscillator.start(0);
  			audioContext.startRendering();
  			audioContext.oncomplete = function (event) {
  				let sum = 0;
  				for (let i = 4500; i < 5000; i++) {
  					sum += Math.abs(event.renderedBuffer.getChannelData(0)[i]);
  				}
  				compressor.disconnect();
  				resolve({
  					key: "audio_fingerprint",
  					value: sum.toString(),
  				});
  			};
  		} catch (error) {
  			resolve(null);
  		}
  	});
  })().then((result) => result?.value);
  ```

  </details>

- Promise:

  ```javascript
  Promise {
    [[Prototype]]: Promise
    [[PromiseState]]: "fulfilled"
    [[PromiseResult]]: "124.04347527516074"
  }
  ```

- Ví dụ: `"124.04347527516074"`

#### Battery Charging Status

- ID: `navigator_battery_charging`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	return new Promise(function (resolve) {
  		if (!navigator.getBattery) {
  			resolve(null);
  			return;
  		}
  		navigator
  			.getBattery()
  			.then(function (battery) {
  				resolve({
  					key: "navigator_battery_charging",
  					value: battery.charging,
  				});
  			})
  			.catch(function () {
  				resolve(null);
  			});
  	});
  })().then((result) => result?.value);
  ```

  </details>

- Promise:

  ```javascript
  Promise {
    [[Prototype]]: Promise
    [[PromiseState]]: "fulfilled"
    [[PromiseResult]]: true
  }
  ```

- Ví dụ: `true`

#### Media Device Kinds

- ID: `media_device_kinds`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (async function () {
  	if (!navigator.mediaDevices || !navigator.mediaDevices.enumerateDevices) {
  		return [];
  	}
  	const deviceKinds = [];
  	const devices = [];
  	try {
  		const mediaDevices = await navigator.mediaDevices.enumerateDevices();
  		for (const device of mediaDevices) {
  			deviceKinds.push(device.kind);
  			const deviceInfo = {
  				kind: device.kind,
  				id: device.deviceId,
  				group: device.groupId,
  			};
  			devices.push(deviceInfo);
  		}
  		const devicesJson = JSON.stringify(devices);
  		return [
  			{
  				key: "media_device_kinds",
  				value: deviceKinds,
  			},
  			{
  				key: "media_devices_hash",
  				value: hashFunctions.md5(devicesJson),
  			},
  		];
  	} catch (error) {
  		return [];
  	}
  })().then((result) => JSON.stringify(result));
  ```

  </details>

- Promise:

  ```javascript
  Promise {
    [[Prototype]]: Promise
    [[PromiseState]]: "fulfilled"
    [[PromiseResult]]: "[{\"key\":\"media_device_kinds\",\"value\":[\"videoinput\",\"audiooutput\"]},{\"key\":\"media_devices_hash\",\"value\":\"a100118c0b7b0da99a7b6db752e59b8c\"}]"
  }
  ```

- Ví dụ: [`"videoinput"`, `"audiooutput"`]

#### Media Devices Hash

- ID: `media_devices_hash`
- Mã:

  > Xem [Media Device Kinds](#media-device-kinds) để biết cách triển khai

- Ví dụ: `"a100118c0b7b0da99a7b6db752e59b8c"`

#### Permissions Hash

- ID: `navigator_permissions_hash`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (async function () {
  	const permissions = ["accelerometer", "accessibility-events", "ambient-light-sensor", "background-sync", "bluetooth", "camera", "clipboard", "clipboard-read", "clipboard-write", "device-info", "geolocation", "gyroscope", "magnetometer", "microphone", "midi", "notifications", "payment-handler", "persistent-storage", "push", "speaker"];
  	if (!navigator?.permissions) {
  		return {
  			key: "navigator_permissions_hash",
  			value: null,
  		};
  	}
  	const supportedPermissions = [];
  	for (const permission of permissions) {
  		try {
  			const result = await navigator.permissions.query({
  				name: permission,
  			});
  			if (result) {
  				supportedPermissions.push(permission);
  			}
  		} catch {
  			continue;
  		}
  	}
  	console.log(supportedPermissions);
  	const hash = hashFunctions.md5(supportedPermissions.join("|"));
  	return {
  		key: "navigator_permissions_hash",
  		value: hash,
  	};
  })().then((result) => result?.value);
  ```

  </details>

- Promise:

  ```javascript
  Promise {
    [[Prototype]]: Promise
    [[PromiseState]]: "fulfilled"
    [[PromiseResult]]: "67419471976a14a1430378465782c62d"
  }
  ```

- Ví dụ: `"67419471976a14a1430378465782c62d"`

#### Math Calculation Hash

- ID: `math_fingerprint`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	var applyMath = function (t) {
  		if (t) {
  			for (var n = arguments.length, r = new Array(n > 1 ? n - 1 : 0), o = 1; o < n; o++) r[o - 1] = arguments[o];
  			return t.apply(undefined, r);
  		}
  		return NaN;
  	};
  	var mathResults = [applyMath(Math.acos, 0.123), applyMath(Math.acosh, Math.SQRT2), applyMath(Math.atan, 2), applyMath(Math.atanh, 0.5), applyMath(Math.cbrt, Math.PI), applyMath(Math.cos, 21 * Math.LN2), applyMath(Math.cos, 21 * Math.SQRT1_2), applyMath(Math.cosh, 492 * Math.LOG2E), applyMath(Math.expm1, 1), applyMath(Math.hypot, Math.LOG2E, -100), applyMath(Math.log10, 7 * Math.LOG10E), applyMath(Math.pow, Math.PI, -100), applyMath(Math.pow, 0.002, -100), applyMath(Math.sin, Math.PI), applyMath(Math.sin, 39 * Math.E), applyMath(Math.sinh, Math.PI), applyMath(Math.sinh, 492 * Math.LOG2E), applyMath(Math.tan, 10 * Math.LOG2E), applyMath(Math.tanh, 0.123)].map(function (t) {
  		return t.toString();
  	});
  	console.log(mathResults);
  	return hashFunctions.md5(mathResults.join(","));
  })();
  ```

  </details>

- Ví dụ: `"3b2ff195f341257a6a2abbc122f4ae67"`

#### Math Function Hash

- ID: `supported_math_functions`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	const mathFunctions = Object.getOwnPropertyNames(Math).filter(function (prop) {
  		return typeof Math[prop] === "function";
  	});
  	console.log(mathFunctions);
  	return hashFunctions.md5(mathFunctions.join(","));
  })();
  ```

  </details>

- Ví dụ: `"e9dd4fafb44ee489f48f7c93d0f48163"`

#### Screen Orientation

- ID: `screen_orientation`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  screen && screen.orientation && screen.orientation.type ? screen.orientation.type : null;
  ```

  </details>

- Ví dụ: `"landscape-primary"`

#### WebRTC Code

- ID: `rtc_peer_connection`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	const rtcPeerConnections = [window.RTCPeerConnection, window.mozRTCPeerConnection, window.webkitRTCPeerConnection];
  	let result = 0;
  	for (let i = 0; i < rtcPeerConnections.length; i++) {
  		if (rtcPeerConnections[i]) {
  			result |= 1 << i;
  		}
  	}
  	return result;
  })();
  ```

  </details>

- Ví dụ: `5`

#### Session UUID

- ID: `4b4b269e68`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  crypto.randomUUID();
  initData ? initData["4b4b269e68"] : null;
  ```

  </details>

- Ví dụ: `"481139d3-f232-49f2-85c0-9f619dea3c93"`

#### Enforcement Hash

- ID: `6a62b2a558`
- Mã:

#### Speech Default Voice

- ID: `speech_default_voice`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	return new Promise(function (resolve) {
  		function processVoices(voices) {
  			var defaultVoice = null;
  			var voicesHash = null;
  			if (voices && voices.length > 0) {
  				var voiceList = voices.reduce(function (acc, voice) {
  					if (voice.default) {
  						defaultVoice = voice.name + " || " + voice.lang;
  					}
  					return acc.concat([[voice.name, voice.lang]]);
  				}, []);
  				if (voiceList.length) {
  					voicesHash = hashFunctions.md5(voiceList.join(","));
  				}
  			}
  			return JSON.stringify([
  				{
  					key: "speech_default_voice",
  					value: defaultVoice,
  				},
  				{
  					key: "speech_voices_hash",
  					value: voicesHash,
  				},
  			]);
  		}
  		try {
  			if (!window.speechSynthesis || !window.speechSynthesis.getVoices || typeof window.speechSynthesis.getVoices != "function") {
  				return resolve(null);
  			}
  			var currentVoices = window.speechSynthesis.getVoices();
  			if (currentVoices.length) {
  				return resolve(processVoices(currentVoices));
  			}
  			window.speechSynthesis.addEventListener("voiceschanged", function handler() {
  				window.speechSynthesis.removeEventListener("voiceschanged", handler);
  				resolve(processVoices(window.speechSynthesis.getVoices()));
  			});
  		} catch (e) {
  			resolve(null);
  		}
  	});
  })();
  ```

  </details>

- Ví dụ: `"Microsoft Sarah Mobile - English (United Kingdom) || en-GB"`
- Promise:

  ```javascript
  Promise {
    [[Prototype]]: Promise
    [[PromiseState]]: "fulfilled"
    [[PromiseResult]]: "[{\"key\":\"speech_default_voice\",\"value\":\"Microsoft Sarah Mobile - English (United Kingdom) || en-GB\"},{\"key\":\"speech_voices_hash\",\"value\":\"6246e0087ff9c2c654100852a6725caf\"}]"
  }
  ```

#### Speech Voices Hash

- ID: `speech_voices_hash`
- Mã:

  > Xem [Speech Default Voice](#speech-default-voice) để biết cách triển khai

- Ví dụ: `"b93c46af83272d759dee3215b52e1698"`

#### Is Keyless

- ID: `is_keyless`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	return !config.publicKey;
  })();
  ```

  </details>

- Ví dụ: `false`

#### Mouse Events

- ID: `4ca87df3d1`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	const mouseTracker = {
  		timestamp: Date.now(),
  		"4ca87df3d1": [],
  	};

  	function insertEvent(e, m) {
  		const event = {
  			timestamp: Date.now() - mouseTracker.timestamp,
  			type: e,
  			x: m.pageX,
  			y: m.pageY,
  		};
  		mouseTracker["4ca87df3d1"].push(event);
  		return event;
  	}

  	function trackMouseMovement(e, m) {
  		if (mouseTracker["4ca87df3d1"].length >= 75) return;

  		if (e === 0) {
  			const lastMove = mouseTracker["4ca87df3d1"][mouseTracker["4ca87df3d1"].length - 1];
  			if (!lastMove) {
  				insertEvent(e, m);
  				return;
  			}
  			const distance = Math.sqrt(Math.pow(m.pageX - lastMove.x, 2) + Math.pow(m.pageY - lastMove.y, 2));
  			if (distance > 5) {
  				insertEvent(e, m);
  			}
  			return;
  		}

  		mouseTracker["4ca87df3d1"].push({
  			timestamp: Date.now() - mouseTracker.timestamp,
  			type: e,
  			x: m.pageX,
  			y: m.pageY,
  		});
  	}

  	// Ví dụ sử dụng:
  	// Mô phỏng sự kiện di chuyển chuột
  	trackMouseMovement(0, { pageX: 100, pageY: 200 });
  	// Mô phỏng di chuyển chuột khác
  	trackMouseMovement(0, { pageX: 150, pageY: 250 });

  	const mouseEvents = mouseTracker["4ca87df3d1"].map((e) => `${e.timestamp},${e.type},${e.x},${e.y}`).join(";");
  	console.log(mouseEvents);
  	return btoa(mouseEvents);
  })();
  ```

  </details>

- Ví dụ: `"MCwwLDEwMCwyMDA7MCwwLDE1MCwyNTA="` // Chuỗi mã hóa Base64 của "0,0,100,200;0,0,150,250"

#### Touch Events

- ID: `867e25e5d4`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	const touchTracker = {
  		timestamp: Date.now(),
  		"867e25e5d4": [],
  	};

  	function trackTouchEvents(v, e) {
  		for (let i = 0; i < v.touches.length; i++) {
  			if (touchTracker["867e25e5d4"].length < 75) {
  				touchTracker["867e25e5d4"].push({
  					timestamp: Date.now() - touchTracker.timestamp,
  					type: e,
  					x: Math.floor(v.touches[i].pageX),
  					y: Math.floor(v.touches[i].pageY),
  				});
  			}
  		}
  		const touchEvents = touchTracker["867e25e5d4"].map((event) => `${event.timestamp},${event.type},${event.x},${event.y}`);
  		console.log(touchEvents);
  		return btoa(touchEvents.join(";"));
  	}

  	// Ví dụ sử dụng:
  	const mockTouchEvent = {
  		touches: [{ pageX: 100, pageY: 200 }],
  	};
  	return trackTouchEvents(mockTouchEvent, 0);
  })();
  ```

  </details>

- Ví dụ: `"MCwwLDEwMCwyMDA="` // Chuỗi mã hóa Base64 của "0,0,100,200"

#### Keyboard Events

- ID: `d4a306884c`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	const keyboardTracker = {
  		timestamp: Date.now(),
  		d4a306884c: [],
  	};

  	const keyboardEventTypes = {
  		Tab: 0,
  		Enter: 1,
  		Space: 3,
  		ShiftLeft: 4,
  		ShiftRight: 5,
  		ControlLeft: 6,
  		ControlRight: 7,
  		MetaLeft: 8,
  		MetaRight: 9,
  		AltLeft: 10,
  		AltRight: 11,
  		Backspace: 12,
  		Escape: 13,
  	};

  	function trackKeyboardEvent(event) {
  		if (keyboardTracker["d4a306884c"].length < 75) {
  			keyboardTracker["d4a306884c"].push({
  				timestamp: Date.now() - keyboardTracker.timestamp,
  				type: event.type,
  				code: keyboardEventTypes[event.code] ?? 14,
  			});
  		}

  		const keyboardEvents = keyboardTracker["d4a306884c"].map((e) => `${e.timestamp},${e.type},${e.code}`);
  		console.log(keyboardEvents);
  		return btoa(keyboardEvents.join(";"));
  	}

  	// Ví dụ sử dụng:
  	const mockKeyEvent = {
  		type: "keydown",
  		code: "Enter",
  	};
  	return trackKeyboardEvent(mockKeyEvent);
  })();
  ```

  </details>

- Ví dụ: `"MCxrZXlkb3duLDE="` // Chuỗi mã hóa Base64 của "0,keydown,1"

### Features

- ID: `fe`
- Mô tả: Các thuộc tính trình duyệt cơ bản

#### Do Not Track

- ID: `DNT`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  navigator.doNotTrack ? navigator.doNotTrack : navigator.msDoNotTrack ? navigator.msDoNotTrack : window.doNotTrack ? window.doNotTrack : "unknown";
  ```

  </details>

- Ví dụ: `"unknown"`

#### Browser Language

- ID: `L`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  navigator.language || navigator.userLanguage || navigator.browserLanguage || navigator.systemLanguage || "";
  ```

  </details>

- Ví dụ: `"en-GB"`

#### Color Depth

- ID: `D`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  screen.colorDepth || -1;
  ```

  </details>

- Ví dụ: `24`

#### Pixel Ratio

- ID: `PR`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  window.devicePixelRatio || "";
  ```

  </details>

- Ví dụ: `1`

#### Screen Size

- ID: `S`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (screen.height > screen.width ? [screen.height, screen.width] : [screen.width, screen.height]).join(",");
  ```

  </details>

- Ví dụ: `"2560,1440"`

#### Available Screen

- ID: `AS`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (screen.availHeight > screen.availWidth ? [screen.availHeight, screen.availWidth] : [screen.availWidth, screen.availHeight]).join(",");
  ```

  </details>

- Ví dụ: `"2560,1392"`

#### Timezone Offset

- ID: `TO`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  new Date().getTimezoneOffset();
  ```

  </details>

- Ví dụ: `-420`

#### Session Storage

- ID: `SS`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	try {
  		return !!window.sessionStorage;
  	} catch (t) {
  		return true;
  	}
  })();
  ```

  </details>

- Ví dụ: `true`

#### Local Storage

- ID: `LS`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	try {
  		return !!window.localStorage;
  	} catch (t) {
  		return true;
  	}
  })();
  ```

  </details>

- Ví dụ: `true`

#### IndexedDB

- ID: `IDB`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	try {
  		return !!window.indexedDB;
  	} catch (t) {
  		return true;
  	}
  })();
  ```

  </details>

- Ví dụ: `true`

#### Add Behavior

- ID: `B`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  !(!document.body || !document.body.addBehavior);
  ```

  </details>

- Ví dụ: `false`

#### Open Database

- ID: `ODB`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  !!window.openDatabase;
  ```

  </details>

- Ví dụ: `false`

#### CPU Class

- ID: `CPUC`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  navigator.cpuClass ? navigator.cpuClass : "unknown";
  ```

  </details>

- Ví dụ: `"unknown"`

#### Platform

- ID: `PK`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  navigator.platform ? navigator.platform : "unknown";
  ```

  </details>

- Ví dụ: `"Win32"`

#### Canvas Fingerprint

- ID: `CFP`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	const canvas = document.createElement("canvas");
  	if (!canvas.getContext) return false;

  	try {
  		const dataPoints = [];
  		canvas.width = 2000;
  		canvas.height = 200;
  		canvas.style.display = "inline";

  		const ctx = canvas.getContext("2d");
  		if (!ctx) return false;

  		// Kiểm tra hướng đường dẫn
  		ctx.rect(0, 0, 10, 10);
  		ctx.rect(2, 2, 6, 6);
  		dataPoints.push(`canvas winding:${ctx.isPointInPath(5, 5, "evenodd") === false ? "yes" : "no"}`);

  		// Vẽ văn bản
  		ctx.textBaseline = "alphabetic";
  		ctx.fillStyle = "#f60";
  		ctx.fillRect(125, 1, 62, 20);
  		ctx.fillStyle = "#069";
  		ctx.font = "11pt no-real-font-123";
  		ctx.fillText("Cwm fjordbank glyphs vext quiz, 😃", 2, 15);
  		ctx.fillStyle = "rgba(102, 204, 0, 0.2)";
  		ctx.font = "18pt Arial";
  		ctx.fillText("Cwm fjordbank glyphs vext quiz, 😃", 4, 45);

  		// Vẽ các vòng tròn chồng lên nhau
  		ctx.globalCompositeOperation = "multiply";
  		ctx.fillStyle = "rgb(255,0,255)";
  		ctx.beginPath();
  		ctx.arc(50, 50, 50, 0, 2 * Math.PI, true);
  		ctx.closePath();
  		ctx.fill();

  		ctx.fillStyle = "rgb(0,255,255)";
  		ctx.beginPath();
  		ctx.arc(100, 50, 50, 0, 2 * Math.PI, true);
  		ctx.closePath();
  		ctx.fill();

  		ctx.fillStyle = "rgb(255,255,0)";
  		ctx.beginPath();
  		ctx.arc(75, 100, 50, 0, 2 * Math.PI, true);
  		ctx.closePath();
  		ctx.fill();

  		// Vẽ các vòng tròn đồng tâm
  		ctx.fillStyle = "rgb(255,0,255)";
  		ctx.arc(75, 75, 75, 0, 2 * Math.PI, true);
  		ctx.arc(75, 75, 25, 0, 2 * Math.PI, true);
  		ctx.fill("evenodd");

  		// Tạo vân tay
  		dataPoints.push(`canvas fp:${canvas.toDataURL()}`);
  		return hashFunctions.simpleHash(dataPoints.join("~"));
  	} catch (error) {
  		return false;
  	}
  })();
  ```

  </details>

- Ví dụ: `-300284282`

#### Font Regular

- ID: `FR`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	const maxScreenDimension = Math.max(screen.width, screen.height);
  	const minScreenDimension = Math.min(screen.width, screen.height);
  	const maxAvailDimension = Math.max(screen.availWidth, screen.availHeight);
  	const minAvailDimension = Math.min(screen.availWidth, screen.availHeight);
  	return maxScreenDimension < maxAvailDimension || minScreenDimension < minAvailDimension;
  })();
  ```

  </details>

- Ví dụ: `false`

#### Font OS

- ID: `FOS`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	const userAgent = navigator.userAgent.toLowerCase();
  	const oscpu = navigator.oscpu;
  	const platform = navigator.platform.toLowerCase();

  	let detectedOS = userAgent.indexOf("android") >= 0 ? "Android" : userAgent.indexOf("windows phone") >= 0 ? "Windows Phone" : userAgent.indexOf("win") >= 0 ? "Windows" : userAgent.indexOf("cros") >= 0 ? "CrOS" : userAgent.indexOf("linux") >= 0 ? "Linux" : userAgent.indexOf("iphone") >= 0 || userAgent.indexOf("ipad") >= 0 || userAgent.indexOf("ipod") >= 0 ? "iOS" : userAgent.indexOf("mac") >= 0 ? "Mac" : "Other";

  	if (typeof oscpu !== "undefined") {
  		const oscpuLower = oscpu.toLowerCase();

  		if ((oscpuLower.indexOf("win") >= 0 && detectedOS !== "Windows" && detectedOS !== "Windows Phone") || (oscpuLower.indexOf("linux") >= 0 && detectedOS !== "Linux" && detectedOS !== "Android") || (oscpuLower.indexOf("mac") >= 0 && detectedOS !== "Mac" && detectedOS !== "iOS") || (oscpuLower.indexOf("win") === 0 && oscpuLower.indexOf("linux") === 0 && oscpuLower.indexOf("mac") >= 0 && detectedOS !== "other")) {
  			return true;
  		}
  	}

  	if (platform.indexOf("win") >= 0 && detectedOS !== "Windows" && detectedOS !== "Windows Phone") {
  		return !(userAgent.indexOf("eawebkit") >= 0);
  	}

  	return ((platform.indexOf("linux") >= 0 || platform.indexOf("android") >= 0 || platform.indexOf("pike") >= 0) && detectedOS !== "Linux" && detectedOS !== "Android" && detectedOS !== "CrOS") || ((platform.indexOf("mac") >= 0 || platform.indexOf("ipad") >= 0 || platform.indexOf("ipod") >= 0 || platform.indexOf("iphone") >= 0) && detectedOS !== "Mac" && detectedOS !== "iOS") || (platform.indexOf("win") === 0 && platform.indexOf("linux") === 0 && platform.indexOf("mac") >= 0 && detectedOS !== "other") || (typeof navigator.plugins === "undefined" && detectedOS !== "Windows" && detectedOS !== "Windows Phone");
  })();
  ```

  </details>

- Ví dụ: `false`

#### Font Browser

- ID: `FB`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	let detectedBrowser;
  	const userAgent = navigator.userAgent.toLowerCase();
  	const productSub = navigator.productSub;

  	detectedBrowser = userAgent.indexOf("firefox") >= 0 ? "Firefox" : userAgent.indexOf("opera") >= 0 || userAgent.indexOf("opr") >= 0 ? "Opera" : userAgent.indexOf("chrome") >= 0 ? "Chrome" : userAgent.indexOf("safari") >= 0 ? "Safari" : userAgent.indexOf("trident") >= 0 ? "Internet Explorer" : "Other";

  	if ((detectedBrowser === "Chrome" || detectedBrowser === "Safari" || detectedBrowser === "Opera") && productSub !== "20030107") {
  		return true;
  	}

  	const evalLength = eval.toString().length;
  	let hasToSource;

  	if (evalLength === 37 && detectedBrowser !== "Safari" && detectedBrowser !== "Firefox" && detectedBrowser !== "Other") {
  		return true;
  	}
  	if (evalLength === 39 && detectedBrowser !== "Internet Explorer" && detectedBrowser !== "Other") {
  		return true;
  	}
  	if (evalLength === 33 && detectedBrowser !== "Chrome" && detectedBrowser !== "Opera" && detectedBrowser !== "Other") {
  		return true;
  	}

  	try {
  		throw "a";
  	} catch (error) {
  		try {
  			error.toSource();
  			hasToSource = true;
  		} catch (e) {
  			hasToSource = false;
  		}
  	}

  	return !(!hasToSource || detectedBrowser === "Firefox" || detectedBrowser === "Other");
  })();
  ```

  </details>

- Ví dụ: `false`

#### JavaScript Fonts

- ID: `JSF`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	if (!document.body) return false;

  	const fontList = [
  		"Andale Mono",
  		"Arial",
  		"Arial Black",
  		"Arial Hebrew",
  		"Arial MT",
  		"Arial Narrow",
  		"Arial Rounded MT Bold",
  		"Arial Unicode MS",
  		"Bitstream Vera Sans Mono",
  		"Book Antiqua",
  		"Bookman Old Style",
  		"Calibri",
  		"Cambria",
  		"Cambria Math",
  		"Century",
  		"Century Gothic",
  		"Century Schoolbook",
  		"Comic Sans",
  		"Comic Sans MS",
  		"Consolas",
  		"Courier",
  		"Courier New",
  		"Garamond",
  		"Geneva",
  		"Georgia",
  		"Helvetica",
  		"Helvetica Neue",
  		"Impact",
  		"Lucida Bright",
  		"Lucida Calligraphy",
  		"Lucida Console",
  		"Lucida Fax",
  		"LUCIDA GRANDE",
  		"Lucida Handwriting",
  		"Lucida Sans",
  		"Lucida Sans Typewriter",
  		"Lucida Sans Unicode",
  		"Microsoft Sans Serif",
  		"Monaco",
  		"Monotype Corsiva",
  		"MS Gothic",
  		"MS Outlook",
  		"MS PGothic",
  		"MS Reference Sans Serif",
  		"MS Sans Serif",
  		"MS Serif",
  		"MYRIAD",
  		"MYRIAD PRO",
  		"Palatino",
  		"Palatino Linotype",
  		"Segoe Print",
  		"Segoe Script",
  		"Segoe UI",
  		"Segoe UI Light",
  		"Segoe UI Semibold",
  		"Segoe UI Symbol",
  		"Tahoma",
  		"Times",
  		"Times New Roman",
  		"Times New Roman PS",
  		"Trebuchet MS",
  		"Verdana",
  		"Wingdings",
  		"Wingdings 2",
  		"Wingdings 3",
  	];

  	const testString = "mmmmmmmmmmlli";
  	const baseFonts = ["monospace", "sans-serif", "serif"];

  	const setFontFamily = (font) => (element) => {
  		const width = element.getAttribute("data-width");
  		element.style.fontFamily = `"${font}", ${width}`;
  	};

  	const setup = (() => {
  		const style = document.createElement("style");
  		style.textContent = `
        .font-parent {
          position: absolute;
          top: 0;
          left: 0;
          visibility: hidden;
        }
        .font {
          font-size: 72px;
          position: absolute;
          left: -9999px;
          line-height: normal;
        }
        ${baseFonts.map((font) => `.font[data-width='${font}'] { font-family: ${font}; }`).join("\n")}
      `;

  		document.head.appendChild(style);

  		const container = document.createElement("div");
  		container.classList.add("font-parent");

  		const spans = [...baseFonts.map((font) => `<span class="font" data-width="${font}">${testString}</span>`), ...baseFonts.map((font, i) => `<span class="font" data-index="${i}" data-width="${font}">${testString}</span>`)].join("\n");

  		container.innerHTML = spans;

  		return {
  			parent: container,
  			cleanup: () => {
  				document.head.removeChild(style);
  				document.body.removeChild(container);
  			},
  		};
  	})();

  	const { parent, cleanup } = setup;
  	document.body.appendChild(parent);

  	const elements = Array.from(parent.children);
  	const baselines = elements.slice(0, 3).map((el) => ({
  		offsetWidth: el.offsetWidth,
  		offsetHeight: el.offsetHeight,
  	}));

  	const testElements = elements.slice(3);
  	const detectedFonts = [];

  	const hasDifferentSize = (font, index) => {
  		return testElements[index].offsetWidth !== baselines[index].offsetWidth || testElements[index].offsetHeight !== baselines[index].offsetHeight;
  	};

  	for (const font of fontList) {
  		testElements.forEach(setFontFamily(font));
  		if (baseFonts.some(hasDifferentSize)) {
  			detectedFonts.push(font);
  		}
  	}

  	cleanup();
  	return detectedFonts.join(",");
  })();
  ```

  </details>

- Ví dụ: `"Arial,Arial Black,Arial Narrow,Book Antiqua,Bookman Old Style,Calibri,Cambria,Cambria Math,Century,Century Gothic,Comic Sans MS,Consolas,Courier,Courier New,Garamond,Georgia,Helvetica,Impact,Lucida Console,Lucida Sans Unicode,Microsoft Sans Serif,Monotype Corsiva,MS Gothic,MS PGothic,MS Reference Sans Serif,MS Sans Serif,MS Serif,Palatino Linotype,Segoe Print,Segoe Script,Segoe UI,Segoe UI Light,Segoe UI Semibold,Segoe UI Symbol,Tahoma,Times,Times New Roman,Trebuchet MS,Verdana,Wingdings,Wingdings 2,Wingdings 3"`

#### Plugins

- ID: `P`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	// Kiểm tra trình duyệt IE/Trident
  	if ((navigator.appName === "Microsoft Internet Explorer" || (navigator.appName === "Netscape" && /Trident/.test(navigator.userAgent))) && ((Object.getOwnPropertyDescriptor && Object.getOwnPropertyDescriptor(window, "ActiveXObject")) || "ActiveXObject" in window)) {
  		// Phát hiện plugin ActiveX cho IE
  		const activeXPlugins = ["AcroPDF.PDF", "Adodb.Stream", "AgControl.AgControl", "DevalVRXCtrl.DevalVRXCtrl.1", "MacromediaFlashPaper.MacromediaFlashPaper", "Msxml2.DOMDocument", "Msxml2.XMLHTTP", "PDF.PdfCtrl", "QuickTime.QuickTime", "QuickTimeCheckObject.QuickTimeCheck.1", "RealPlayer", "RealPlayer.RealPlayer(tm) ActiveX Control (32-bit)", "RealVideo.RealVideo(tm) ActiveX Control (32-bit)", "Scripting.Dictionary", "SWCtl.SWCtl", "Shell.UIHelper", "ShockwaveFlash.ShockwaveFlash", "Skype.Detection", "TDCCtl.TDCCtl", "WMPlayer.OCX", "rmocx.RealPlayer G2 Control", "rmocx.RealPlayer G2 Control.1"].reduce((detectedPlugins, plugin) => {
  			try {
  				new ActiveXObject(plugin);
  				return [...detectedPlugins, plugin];
  			} catch (e) {
  				return detectedPlugins;
  			}
  		}, []);

  		return activeXPlugins;
  	}

  	// Phát hiện plugin tiêu chuẩn cho các trình duyệt khác
  	const plugins = [];
  	if (navigator.plugins) {
  		for (let i = 0; i < navigator.plugins.length; i++) {
  			const plugin = navigator.plugins[i];
  			if (plugin && plugin.name) {
  				plugins.push(plugin.name);
  			}
  		}
  	}
  	return plugins.sort().join(",");
  })();
  ```

  </details>

- Ví dụ: `"Chromium PDF Plugin,Chromium PDF Viewer"`

#### Touch

- ID: `T`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	let touchPoints = 0;
  	let hasTouchEvent = false;

  	if (typeof navigator.maxTouchPoints !== "undefined") {
  		touchPoints = navigator.maxTouchPoints;
  	} else if (typeof navigator.msMaxTouchPoints !== "undefined") {
  		touchPoints = navigator.msMaxTouchPoints;
  	}

  	if (isNaN(touchPoints)) {
  		touchPoints = -999;
  	}

  	try {
  		document.createEvent("TouchEvent");
  		hasTouchEvent = true;
  	} catch (error) {}

  	return [touchPoints, hasTouchEvent, "ontouchstart" in window].join(",");
  })();
  ```

  </details>

- Ví dụ: `"0,false,false"`

#### Hardware Concurrency

- ID: `H`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  navigator.hardwareConcurrency ? navigator.hardwareConcurrency : "unknown";
  ```

  </details>

- Ví dụ: `32`

#### SWF

- ID: `SWF`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  typeof window.swfobject !== "undefined";
  ```

  </details>

- Ví dụ: `false`

### Feature Items Hash

- ID: `ife_hash`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  hashFunctions.murmurHash3(fe_items.join(";"), 0);
  ```

  </details>

- Ví dụ: `"7883868bba11a9bd8ee6e3fa1bad8e4f"`

### JavaScript Behavior Data

- ID: `jsbd`
- Mã:

  <details>
  <summary>Xem mã</summary>

  ```javascript
  (function () {
  	let webdriverValue = JSON.stringify(navigator.webdriver);
  	if (navigator.webdriver === undefined) {
  		webdriverValue = "undefined";
  		if (Object.getOwnPropertyDescriptor(navigator, "webdriver")) {
  			webdriverValue = "faked";
  		}
  	}

  	// HL: Độ dài lịch sử - số lượng mục trong lịch sử trình duyệt
  	// NCE: Cookie trình duyệt được bật - kiểm tra cookie có được bật hay không
  	// DT: Tiêu đề tài liệu - tiêu đề trang hiện tại
  	// NWD: WebDriver trình duyệt - trạng thái webdriver (true/false/undefined/faked)
  	// DMTO: DOM Mutation Observer - luôn là 1
  	// DOTO: DOM Object Type Observer - luôn là 1
  	const data = {
  		HL: window.history.length,
  		NCE: navigator.cookieEnabled,
  		DT: document.title,
  		NWD: webdriverValue,
  		DMTO: 1,
  		DOTO: 1,
  	};

  	return JSON.stringify(data);
  })();
  ```

  </details>

- Ví dụ:

  - Dữ liệu gốc:

    ```json
    {
    	"HL": 7,
    	"NCE": true,
    	"DT": "",
    	"NWD": "false",
    	"DMTO": 1,
    	"DOTO": 1
    }
    ```

  - Chuỗi JSON sau khi mã hóa:

    ```json
    "{\"HL\":7,\"NCE\":true,\"DT\":\"\",\"NWD\":\"false\",\"DMTO\":1,\"DOTO\":1}"
    ```
