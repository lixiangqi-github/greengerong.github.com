---
layout: post
title: "生成PDF的新选择-Phantomjs"
description: ""
category: "phantomjs"
tags: [phantomjs,javascript]
---

最近在node.js项目开发中，遇见生成PDF的需求，当然生成PDF不是一个新意的需求；我可以选择利用开源的pdfkit或者其他node pdf模块，或者通过edge.js调用.net/python下的pdf库去做生成pdf。但是在我看来对于这些东西不管如何也需要花费我们太多的时间(pdf报表的内容报表很复杂)，不如把所有的画图实现逻辑推向大家所熟悉的html+css来的简洁，快速，这样对于pdf格式变化和图形计算逻辑的变化推到ejs、jade之类的模板引擎，对于以后的修改维护扩展是个很不错的选择。所以选择phantomjs加载页面生成PDF对于我来说不是个不错的选择，同时对于html+css我所需要兼容的仅有webkit一种浏览器，没有厌恶的浏览器兼容性顾虑。所以说做就做，我在项目上花了半个小时配置phantomjs的自动化脚本（在各环境能够自动勾践），以及实现了一个简单页面的PDF转化。

rasterize.js（来自官方pdf demo）：

	var page = require('webpage').create(),
		    system = require('system'),
		    address, output, size;

		if (system.args.length < 3 || system.args.length > 5) {
		    console.log('Usage: rasterize.js URL filename [paperwidth*paperheight|paperformat] [zoom]');
		    console.log('  paper (pdf output) examples: "5in*7.5in", "10cm*20cm", "A4", "Letter"');
		    phantom.exit(1);
		} else {
		    address = system.args[1];
		    output = system.args[2];
		    page.viewportSize = { width: 600, height: 600 };
		    if (system.args.length > 3 && system.args[2].substr(-4) === ".pdf") {
		        size = system.args[3].split('*');
		        page.paperSize = size.length === 2 ? { width: size[0], height: size[1], margin: '0px' }
		            : { format: system.args[3], orientation: 'portrait', margin: '1cm' };
		    }
		    if (system.args.length > 4) {
		        page.zoomFactor = system.args[4];
		    }
		    page.open(address, function (status) {
		        if (status !== 'success') {
		            console.log('Unable to load the address!');
		            phantom.exit();
		        } else {
		            window.setTimeout(function () {
		                page.render(output);
		                phantom.exit();
		            });
		        }
		    });
		}
		
		
在node调用端，使用exec调用命令行输入得到文件并返回到node response流：


guid utils:


		'use strict';

		var guid = function () {
		    var uid = 0;
		    this.newId = function () {
		        uid = uid % 1000;
		        var now = new Date();
		        var utc = new Date(now.getTime() + now.getTimezoneOffset() * 60000);
		        return utc.getTime() + uid++;
		    }
		}

		exports.utils = {
		    guid: new guid()
		};

pdfutil:

		'use strict';

		var exec = require('child_process').exec;
		var utils = require('./utils').utils;
		var nodeUtil = require('util');

		var outPut = function (id, req, res) {
		    var path = nodeUtil.format("tmp/%s.pdf", utils.guid.newId());
		    var port = req.app.settings.port;
		    var pdfUrl = nodeUtil.format("%s://%s%s/pdf/%s", req.protocol, req.host, ( port == 80 || port == 443 ? '' : ':' + port ), id);

		    exec(nodeUtil.format("phantomjs tool/rasterize.js %s %s A4", pdfUrl, path), function (error, stdout, stderr) {
		        if (error || stderr) {
		            res.send(500, error || stderr);
		            return;
		        }
		        res.set('Content-Type', 'application/pdf');
		        res.download(path);
		    });

		};

		exports.pdfUtils = {
		    outPut: outPut
		};


响应的代码也可以很好的转换为java/c#...的命令行调用来得到pdf并推送到response流中。一切都这么简单搞定。

node也有node-phantom模块，但是用它生成的pdf样式有点怪，所以最后还是坚持采用了exec方式去做。

还有就是phantomjs生成PDF不会把css的背景色和北京图片带进去，所以对于这块专门利用了纯色图片img标签，并position:relative或者absolute去定位文字.这点还好因为这个页面上用户不会看的，

文章也到此结尾，希望多多交流，继续关注，谢谢大家。