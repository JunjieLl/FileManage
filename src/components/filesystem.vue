<template>
  <div>
    <el-container>
      <el-header>
        <el-row>
          <el-col :span="2"><el-tag>1850668 李俊杰</el-tag> </el-col>
          <el-col :span="2"><el-tag>文件管理系统</el-tag> </el-col>
          <el-col :span="2"
            ><el-tag>空闲块指针{{ freeBlock }}</el-tag>
          </el-col>
          <el-col :span="2"
            ><el-tag>路径:/{{ PATH }}</el-tag>
          </el-col>
          <el-col :span="1" :offset="8">
            <el-button type="primary" @click="init">格式化</el-button>
          </el-col>
          <el-col :span="1" :offset="1"
            ><el-button
              type="primary"
              icon="el-icon-back"
              @click="back"
            ></el-button
          ></el-col>
          <el-col :span="1">
            <el-button
              type="primary"
              icon="el-icon-folder-add"
              @click="addSubDirectory(input)"
            ></el-button>
          </el-col>
          <el-col :span="1">
            <el-button
              type="primary"
              icon="el-icon-document-add"
              @click="addFile(input)"
            ></el-button>
          </el-col>
          <el-col :span="3">
            <el-input
              v-model="input"
              placeholder="请输入要创建的文件名"
            ></el-input>
          </el-col>
        </el-row>
      </el-header>
      <el-container>
        <el-aside>
          <el-row>磁盘块</el-row>
          <el-table :data="block" stripe style="width: 100%" height="700px">
            <el-table-column prop="blockNo" label="编号"> </el-table-column>
            <el-table-column prop="nextBlock" label="链接"> </el-table-column>
          </el-table>
        </el-aside>
        <el-container>
          <el-main>
            <el-dialog
              :title="openFileName"
              :visible.sync="dialogTableVisible"
              @close="save"
            >
              <el-row>
                <el-col :span="1" :offset="20">
                  <el-button type="text" @click="save">保存</el-button>
                </el-col>
                <el-col :span="1" :offset="1">
                  <el-button type="text" @click="dialogTableVisible = false"
                    >关闭</el-button
                  >
                </el-col>
              </el-row>
              <el-row>
                <el-input
                  type="textarea"
                  autosize
                  placeholder="文件为空"
                  v-model="textarea"
                >
                </el-input>
              </el-row>
            </el-dialog>

            <el-table
              :data="currentDirectoryList"
              style="width: 100%"
              empty-text="文件夹为空"
              height="700px"
            >
              <el-table-column label="文件名">
                <template slot-scope="scope">
                  <el-tag
                    :class="
                      scope.row.isDirectory
                        ? 'el-icon-folder'
                        : 'el-icon-document'
                    "
                  >
                    {{ scope.row.filename }}
                  </el-tag>
                </template>
              </el-table-column>
              <el-table-column label="种类">
                <template slot-scope="scope">
                  <span>{{ scope.row.isDirectory ? "文件夹" : "文件" }}</span>
                </template>
              </el-table-column>
              <el-table-column prop="createTime" label="创建日期">
              </el-table-column>
              <el-table-column prop="latestModifiedTime" label="修改日期">
              </el-table-column>
              <el-table-column prop="length" label="长度"> </el-table-column>
              <el-table-column prop="startPos" label="物理地址">
              </el-table-column>
              <el-table-column label="操作">
                <template slot-scope="scope">
                  <el-button size="mini" @click="handleOpen(scope.row)"
                    >打开</el-button
                  >
                  <el-button
                    size="mini"
                    type="danger"
                    @click="handleDelete(scope.row.filename)"
                    >删除</el-button
                  >
                </template>
              </el-table-column>
            </el-table>
          </el-main>
        </el-container>
      </el-container>
    </el-container>
  </div>
</template>

<script>
export default {
  name: "Home",
  data() {
    Date.prototype.format = function (fmt) {
      var o = {
        "M+": this.getMonth() + 1, //月份
        "d+": this.getDate(), //日
        "h+": this.getHours(), //小时
        "m+": this.getMinutes(), //分
        "s+": this.getSeconds(), //秒
        "q+": Math.floor((this.getMonth() + 3) / 3), //季度
        S: this.getMilliseconds(), //毫秒
      };
      if (/(y+)/.test(fmt)) {
        fmt = fmt.replace(
          RegExp.$1,
          (this.getFullYear() + "").substr(4 - RegExp.$1.length)
        );
      }
      for (var k in o) {
        if (new RegExp("(" + k + ")").test(fmt)) {
          fmt = fmt.replace(
            RegExp.$1,
            RegExp.$1.length == 1
              ? o[k]
              : ("00" + o[k]).substr(("" + o[k]).length)
          );
        }
      }
      return fmt;
    };
    var directoryItem = function (
      filename,
      isDirectory,
      startPos,
      length = 1,
      fileindex = new Date().getTime(),
      createTime = new Date().format("yyyy-MM-dd hh:mm:ss"),
      latestModifiedTime = new Date().format("yyyy-MM-dd hh:mm:ss")
    ) {
      this.filename = filename;
      this.isDirectory = isDirectory; //是否是子目录
      this.startPos = startPos; //指向下一级目录的块号或是文件的物理地址
      this.createTime = createTime;
      this.latestModifiedTime = latestModifiedTime;
      this.length = length;
      this.fileIndex = fileindex; //文件索引
    };
    var directory = function () {
      this.directoryList = new Array();
    };
    var block = function (blockNo, nextBlock, blockContent) {
      this.blockNo = blockNo;
      this.content = blockContent;
      this.nextBlock = nextBlock; //链接
    };
    return {
      createDirectoryItem: directoryItem, //1个块装10个item
      createDirectory: directory,
      createBlock: block,

      block: [], //物理块
      rootDirectory: Number, //根目录所在块号
      currentDirectory: Number, //当前所在目录块号
      blockNum: Number, //总共100个块
      blockSize: Number, //每个块content 10KB大
      freeBlock: Number, //首个空闲块的索引

      //路径栈
      pathList: [], //路径,对相应目录项的引用

      //输入条
      input: "",
      textarea: "",
      dialogTableVisible: false,
      openFileName: "",
      openfileItem: "",
    };
  },
  computed: {
    currentDirectoryList: function () {
      var ar = new Array();
      var cur = this.currentDirectory;
      while (cur !== -1) {
        for (var i = 0; i < this.block[cur].content.directoryList.length; i++) {
          ar.push(this.block[cur].content.directoryList[i]);
        }
        cur = this.block[cur].nextBlock;
      }
      return ar;
    },
    PATH: function () {
      var str = "";
      for (var i = 0; i < this.pathList.length; i++) {
        str += this.pathList[i].filename + "/";
      }
      return str;
    },
  },
  created() {
    this.init();
  },
  methods: {
    init: function () {
      //extra initialization
      this.input = "";
      this.textarea = "";
      this.dialogTableVisible = false;
      this.openfileItem = "";
      this.openFileName = "";

      this.pathList = [];
      this.block = [];

      //created
      this.blockNum = 200;
      this.blockSize = 10;
      //初始化磁盘块，存储信息
      for (var i = 0; i < this.blockNum; i++) {
        this.block.push(new this.createBlock(i, i + 1)); //创建物理块
      }
      this.block[this.blockNum - 1].nextBlock = -1; //封闭

      //空闲链表
      this.freeBlock = 0;

      //根目录
      this.rootDirectory = this.allocateFreeBlockForDirectory();
      this.pathList.push(new this.createDirectoryItem("root", true, 0));
      this.currentDirectory = this.rootDirectory;
    },
    back: function () {
      if (this.pathList.length === 1) {
        return; //根目录
      }

      this.pathList.pop();
      this.currentDirectory = this.pathList[this.pathList.length - 1].startPos;
    },
    getFileContent: function (blockNo) {
      var content = "";
      while (blockNo !== -1) {
        content += this.block[blockNo].content;
        blockNo = this.block[blockNo].nextBlock;
      }
      return content;
    },
    handleOpen: function (dirItem) {
      if (dirItem.isDirectory === true) {
        //目录
        this.currentDirectory = dirItem.startPos;
        this.pathList.push(dirItem);
      } else {
        //文件
        this.openFileName = dirItem.filename;
        this.textarea = this.getFileContent(dirItem.startPos);
        this.openfileItem = dirItem;
        this.dialogTableVisible = true;
      }
    },
    save: function () {
      var content = this.getFileContent(this.openfileItem.startPos);
      if (content === this.textarea) {
        return; //无需保存
      } else {
        //将其块回收并进行重分配
        var st = this.openfileItem.startPos;
        this.openfileItem.startPos = -1;
        while (st !== -1) {
          var p = this.block[st].nextBlock;
          this.block[st].nextBlock = this.freeBlock;
          this.freeBlock = st;
          st = p;
        }
        //分配，每块30个字符
        var contentList = new Array(),
          i = 0;
        while (i < this.textarea.length) {
          contentList.push(this.textarea.substr(i, 3 * this.blockSize));
          i += 3 * this.blockSize;
        }
        //更新长度
        this.openfileItem.length = contentList.length;
        this.openfileItem.latestModifiedTime = new Date().format(
          "yyyy-MM-dd hh:mm:ss"
        );
        var cur,
          first = true;
        while (contentList.length !== 0) {
          p = this.allocateFreeBlockFroFile();
          if (p === undefined) {
            alert("无空闲空间分配");
            return;
          }
          if (first === true) {
            this.openfileItem.startPos = p;
            cur = this.openfileItem.startPos;
            first = false;
          } else {
            this.block[cur].nextBlock = p;
            cur = p;
          }
          this.block[cur].content = contentList.shift();
        }
        for (i = 0; i < this.pathList.length; i++) {
          this.pathList[i].latestModifiedTime = new Date().format(
            "yyyy-MM-dd hh:mm:ss"
          );
        }
      }
    },
    getDeleteList: function (directoryBlockNo, delArray) {
      if (directoryBlockNo === -1) {
        return;
      }
      while (directoryBlockNo !== -1) {
        for (
          var i = 0;
          i < this.block[directoryBlockNo].content.directoryList.length;
          i++
        ) {
          if (
            this.block[directoryBlockNo].content.directoryList[i]
              .isDirectory === true
          ) {
            this.getDeleteList(
              this.block[directoryBlockNo].content.directoryList[i].startPos,
              delArray
            );
          }
          delArray.push(
            this.block[directoryBlockNo].content.directoryList[i].startPos
          );
        }
        directoryBlockNo = this.block[directoryBlockNo].nextBlock;
      }
    },
    handleDelete: function (filename) {
      var cur = this.currentDirectory,
        pre = -1;
      //查找该目录条目
      while (cur !== -1) {
        for (var i = 0; i < this.block[cur].content.directoryList.length; i++) {
          if (this.block[cur].content.directoryList[i].filename === filename) {
            //是目录则递归删除
            if (this.block[cur].content.directoryList[i].isDirectory === true) {
              var delArray = new Array();
              delArray.push(this.block[cur].content.directoryList[i].startPos);
              this.getDeleteList(
                this.block[cur].content.directoryList[i].startPos,
                delArray
              ); //获取删除列表
              for (var j = 0; j < delArray.length; j++) {
                var st = delArray[j];
                while (st !== -1) {
                  //将该文件（目录）的所有块释放回收
                  var p = this.block[st].nextBlock;
                  this.block[st].nextBlock = this.freeBlock;
                  this.freeBlock = st;
                  st = p;
                }
              }
            }
            //是文件，直接删除即可
            else {
              st = this.block[cur].content.directoryList[i].startPos;
              while (st !== -1) {
                //将该文件（目录）的所有块释放回收
                p = this.block[st].nextBlock;
                this.block[st].nextBlock = this.freeBlock;
                this.freeBlock = st;
                st = p;
              }
            }
            //删除目录条目
            this.block[cur].content.directoryList.splice(i, 1);
            for (i = 0; i < this.pathList.length; i++) {
              this.pathList[i].latestModifiedTime = new Date().format(
                "yyyy-MM-dd hh:mm:ss"
              );
            }

            if (
              this.block[cur].content.directoryList.length === 0 &&
              cur !== 0
            ) {
              //回收该块
              if (pre !== -1) {
                this.block[pre].nextBlock = this.block[cur].nextBlock;
              } else {
                this.pathList[this.pathList.length - 1].startPos =
                  this.block[cur].nextBlock;
              }
              this.block[cur].nextBlock = this.freeBlock;
              this.freeBlock = cur;
            }
            return;
          }
        }
        pre = cur;
        cur = this.block[cur].nextBlock;
      }
    },
    checkSameName: function (filename) {
      for (var i = 0; i < this.currentDirectoryList.length; i++) {
        if (this.currentDirectoryList[i].filename === filename) {
          return true;
        }
      }
      return false;
    },
    allocateFreeBlockForDirectory: function () {
      if (this.freeBlock !== -1) {
        var free = this.freeBlock;
        this.freeBlock = this.block[this.freeBlock].nextBlock; //空闲块指针后移
        this.block[free].nextBlock = -1; //封闭
        this.block[free].content = new this.createDirectory(); //初始化
        return free;
      }
      alert("无空闲空间可分配");
      return undefined;
    },
    allocateFreeBlockFroFile: function () {
      if (this.freeBlock !== -1) {
        var free = this.freeBlock;
        this.freeBlock = this.block[this.freeBlock].nextBlock; //空闲块指针后移
        this.block[free].nextBlock = -1; //封闭
        this.block[free].content = ""; //初始化,文件内容为文本，每个块能装30个字符
        return free;
      }
      alert("无空闲空间可分配");
      return undefined;
    },
    addFile: function (filename = "新建文件") {
      if (this.checkSameName(filename) === true) {
        alert("同名文件已经创建！");
        return;
      }
      if (filename === "") {
        alert("文件名不能为空！");
        return;
      }
      //文件初始时分配一个空FCB,不需要分配物理块
      // var free = this.allocateFreeBlockFroFile();
      // if (free === undefined) {
      //   return;
      // }
      var free = -1;

      //添加目录项
      if (
        this.block[this.currentDirectory].content.directoryList.length <
        this.blockSize
      ) {
        //还可以添加在当前目录块
        this.block[this.currentDirectory].content.directoryList.push(
          new this.createDirectoryItem(filename, false, free, 0)
        );
      } else {
        //向后寻找看是否有扩展目录
        var pre = this.currentDirectory,
          cur = this.block[this.currentDirectory].nextBlock;
        while (cur !== -1) {
          if (this.block[cur].content.directoryList.length === this.blockSize) {
            pre = cur;
            cur = this.block[cur].nextBlock;
          } else {
            break;
          }
        }

        if (cur !== -1) {
          this.block[cur].content.directoryList.push(
            new this.createDirectoryItem(filename, false, free, 0)
          );
        } else {
          var nextfree = this.allocateFreeBlockForDirectory();
          if (free === undefined) {
            return;
          }
          this.block[pre].nextBlock = nextfree;
          this.block[nextfree].content.directoryList.push(
            new this.createDirectoryItem(filename, true, free, 0)
          );
          this.pathList[this.pathList.length - 1].length++;
        }
      }
      for (var i = 0; i < this.pathList.length; i++) {
        this.pathList[i].latestModifiedTime = new Date().format(
          "yyyy-MM-dd hh:mm:ss"
        );
      }
    },

    addSubDirectory: function (filename = "新建文件夹") {
      if (this.checkSameName(filename) === true) {
        alert("同名文件已经创建！");
        return;
      }
      if (filename === "") {
        alert("文件名不能为空！");
        return;
      }
      //取出一个空闲块分配给该子目录
      var free = this.allocateFreeBlockForDirectory();
      if (free === undefined) {
        return;
      }

      //添加目录项
      if (
        this.block[this.currentDirectory].content.directoryList.length <
        this.blockSize
      ) {
        //还可以添加在当前目录块
        this.block[this.currentDirectory].content.directoryList.push(
          new this.createDirectoryItem(filename, true, free, 1)
        );
      } else {
        //向后寻找看是否有扩展目录
        var pre = this.currentDirectory,
          cur = this.block[this.currentDirectory].nextBlock;
        while (cur !== -1) {
          if (this.block[cur].content.directoryList.length === this.blockSize) {
            pre = cur;
            cur = this.block[cur].nextBlock;
          } else {
            break;
          }
        }

        if (cur !== -1) {
          this.block[cur].content.directoryList.push(
            new this.createDirectoryItem(filename, true, free, 1)
          );
        } else {
          var nextfree = this.allocateFreeBlockForDirectory();
          if (free === undefined) {
            return;
          }
          this.block[pre].nextBlock = nextfree;
          this.block[nextfree].content.directoryList.push(
            new this.createDirectoryItem(filename, true, free, 1)
          );
          this.pathList[this.pathList.length - 1].length++;
        }
      }
      for (var i = 0; i < this.pathList.length; i++) {
        this.pathList[i].latestModifiedTime = new Date().format(
          "yyyy-MM-dd hh:mm:ss"
        );
      }
    },
  },
};
</script>

<style scoped>
.text {
  font-size: 14px;
}

.item {
  margin-bottom: 18px;
}

.clearfix:before,
.clearfix:after {
  display: table;
  content: "";
}
.clearfix:after {
  clear: both;
}

.box-card {
  width: 480px;
}
</style>