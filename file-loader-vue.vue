<template>
   <div>
     <input
       type="file"
       id="inputFile"
       @change="handleChange"
     />
   </div>
 </template>

 <script>
 
 
 
 //以下为汇总的一些文件的二进制标识：

//JPEG/JPG - 文件头标识 (2 bytes): ff, d8 文件结束标识 (2 bytes): ff, d9
//TGA - 未压缩的前 5 字节 00 00 02 00 00 - RLE 压缩的前 5 字节 00 00 10 00 00
//PNG - 文件头标识 (8 bytes) 89 50 4E 47 0D 0A 1A 0A
//GIF - 文件头标识 (6 bytes) 47 49 46 38 39 (37) 61
//BMP - 文件头标识 (2 bytes) 42 4D B M
//PCX - 文件头标识 (1 bytes) 0A
//TIFF - 文件头标识 (2 bytes) 4D 4D 或 49 49
//ICO - 文件头标识 (8 bytes) 00 00 01 00 01 00 20 20
//CUR - 文件头标识 (8 bytes) 00 00 02 00 01 00 20 20
//IFF - 文件头标识 (4 bytes) 46 4F 52 4D
//ANI - 文件头标识 (4 bytes) 52 49 46 46
 
 
 export default {
   name: "file-loader-vue",
   methods: {
		 check(headers) {
		   return (buffers, options = { offset: 0 }) =>
			   headers.every(
			   (header, index) => header === buffers[options.offset + index]
		   );
		 },
		 async handleChange(event) {
		   const file = event.target.files[0];
		   // 以PNG为例，只需要获取前8个字节，即可识别其类型
		   const buffers = await this.readBuffer(file, 0, 8);
		   const uint8Array = new Uint8Array(buffers);
		   const isPNG = this.check([0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a]);
		   // 上传test.png后，打印结果为true
		   console.log(isPNG(uint8Array))
		   
		   
		   if(isPNG(uint8Array)){
		   
				this.createWorker(file,1000)
		   
		   }
		   
		 },
		 readBuffer(file, start = 0, end = 2) {
		   // 获取文件的二进制数据，因为我们只需要校验前几个字节即可，所以并不需要获取整个文件的数据
			 return new Promise((resolve, reject) => {
			   const reader = new FileReader();
			   reader.onload = () => {
				 resolve(reader.result);
			   };
			   reader.onerror = reject;
			   reader.readAsArrayBuffer(file.slice(start, end));
			 });
		 },
		 createWorker(file,DefualtChunkSize){
				// DefualtChunkSize 每个切割单元的尺寸大小
			 // 创建一个worker对象
			 const worker = new worker('worker.js')
			 // 向子线程发送消息，并传入文件对象和切片大小，开始计算分割切片
			 worker.postMessage(file, DefualtChunkSize)

			 // 子线程计算完成后，会将切片返回主线程
			 worker.onmessage = ({hash,chunks}) => {
				this.checkAndUploadChunk(chunks,hash)
			 }
		 
		 },
		 // 检查是否已存在相同文件
		async checkAndUploadChunk(chunkList, fileMd5Value) {
			 const requestList = []
			 // 如果不存在，则上传
			 for (let i = 0; i < chunkList; i++) {
			   requestList.push(this.upload({ chunkList[i], fileMd5Value, i }))
			 }
			 // 并发上传
			 if (requestList?.length) {
			   await Promise.all(requestList)
			 }
	   },
	    // 上传chunk
		upload({ chunkList, chunk, fileMd5Value, i }) {
			 current = 0
			 let form = new FormData()
			 form.append("data", chunk) //切片流
			 form.append("total", chunkList.length) //总片数
			 form.append("index", i) //当前是第几片
			 form.append("fileMd5Value", fileMd5Value)
			 return axios({
				 method: 'post',
				 url: BaseUrl + "/upload",
				 data: form    
				 }).then(({ data }) => {
				 if (data.stat) {
				 current = current + 1
				 // 获取到上传的进度
				 const uploadPercent = Math.ceil((current / chunkList.length) * 100)
			   }
			 })
		}
   }
 };
 </script>