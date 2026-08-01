<html><head><script name="BTRoblox/inject.js">"use strict";
(() => {
	const templates = {}
	let settingsAreLoaded = false
	let gtsNode
 
	let settings
	let currentPage
	let matches
	let IS_DEV_MODE
 
	const ContentJS = {
		send(action, ...args) {
			document.dispatchEvent(new CustomEvent("content." + action, { detail: args }))
		},
		listen(actions, callback, props) {
			const actionList = actions.split(" ")
			const once = props && props.once
 
			const cb = ev => {
				if(once) {
					actionList.forEach(action => {
						document.removeEventListener(`inject.${action}`, cb)
					})
				}
 
				return callback(...ev.detail)
			}
 
			actionList.forEach(action => {
				document.addEventListener(`inject.${action}`, cb)
			})
		}
	}
 
	function HijackAngular(moduleName, objects) {
		try {
			const module = angular.module(moduleName)
			const done = {}
 
			module._invokeQueue.forEach(data => {
				const [, type, data2] = data
				const [name, value] = data2
				const fn = objects[name]
				if(!fn) { return }
 
				done[name] = true
				if(type === "constant" || type === "component") {
					try { fn(value) }
					catch(ex) { console.error(ex) }
 
					return
				}
 
				if(typeof value === "function") {
					const injects = value.$inject
					const oldFn = value
 
					data2[1] = new Proxy(oldFn, {
						apply(target, thisArg, args) {
							const argMap = {}
							args.forEach((x, i) => argMap[injects[i]] = x)
 
							return fn.call(thisArg, target, args, argMap)
						}
					})
				} else {
					const injects = value
					const oldFn = value[value.length - 1]
 
					if(typeof oldFn === "function") {
						value[value.length - 1] = new Proxy(oldFn, {
							apply(target, thisArg, args) {
								const argMap = {}
								args.forEach((x, i) => argMap[injects[i]] = x)
 
								return fn.call(thisArg, target, args, argMap)
							}
						})
					} else {
						done[name] = false
					}
				}
			})
 
			if(IS_DEV_MODE) {
				Object.keys(objects).forEach(name => {
					if(!done[name]) {
						console.warn(`Failed to hijack ${moduleName}.${name}`)
						if(IS_DEV_MODE) { alert(`HijackAngular Missing Module '${moduleName}.${name}'`) }
					}
				})
			}
		} catch(ex) {
			if(IS_DEV_MODE) {
				console.warn(ex)
			}
		}
	}
 
	function PreInit() {
		const onSet = (a, b, c) => {
			if(a[b]) { return c(a[b]) }
 
			Object.defineProperty(a, b, {
				enumerable: false,
				configurable: true,
				set(v) {
					delete a[b]
					a[b] = v
					c(v)
				}
			})
		}
 
		if(window.googletag) {
			if(IS_DEV_MODE) {
				console.warn("[BTRoblox] Failed to load inject before googletag")
			}
		} else {
			onSet(window, "googletag", gtag => onSet(gtag, "cmd", () => {
				let didIt = false
 
				const proto = Node.prototype
				const insertBefore = proto.insertBefore
				proto.insertBefore = function(...args) {
					const node = args[0]
					if(node instanceof Node && node.nodeName === "SCRIPT" && node.src.includes("googletagservices.com")) {
						didIt = true
 
						if(!settingsAreLoaded) {
							gtsNode = { this: this, node }
							return
						} else if(settings.general.hideAds) {
							return
						}
					}
 
					return insertBefore.apply(this, args)
				}
 
				setTimeout(() => {
					proto.insertBefore = insertBefore
 
					if(!didIt && IS_DEV_MODE) {
						alert("Failed to rek googletag")
					}
				}, 0)
			}))
		}
	}
 
	function PostInit() {
		if(gtsNode) {
			if(!settings.general.hideAds) {
				gtsNode.this.insertBefore(gtsNode.node)
			}
 
			gtsNode = null
		}
	}
 
	function DocumentReady() {
		if(!window.jQuery) {
			console.warn("[BTR] window.jQuery not set")
			return
		}
 
		if(window.angular) {
			const templateCache = {}
 
			angular.module("ng").run($templateCache => {
				const put = $templateCache.put
 
				$templateCache.put = (key, value) => {
					let result
 
					if(templates[key]) {
						delete templates[key]
 
						let didReturn = false
						ContentJS.listen(`TEMPLATE_${key}`, changedValue => {
							templateCache[key] = changedValue
 
							result = put.call($templateCache, key, changedValue)
							didReturn = true
						})
 
						ContentJS.send(`TEMPLATE_${key}`, value)
 
						console.assert(didReturn, "Template modified did not return in time")
					} else {
						if(key in templateCache) {
							value = templateCache[key]
						}
 
						result = put.call($templateCache, key, value)
					}
 
					return result
				}
			})
 
			if(settings.general.smallChatButton) {
				HijackAngular("chat", {
					chatController(func, args, argMap) {
						const result = func.apply(this, args)
 
						try {
							const { $scope, chatUtility } = argMap
 
							const library = $scope.chatLibrary
							const width = library.chatLayout.widthOfChat
 
							$scope.$watch(() => library.chatLayout.collapsed, value => {
								library.chatLayout.widthOfChat = value ? 54 + 6 : width
								chatUtility.updateDialogsPosition(library)
							})
						} catch(ex) {
							console.error(ex)
							if(IS_DEV_MODE) { alert("HijackAngular Error") }
						}
 
						return result
					}
				})
			}
 
			if(currentPage === "inventory" && settings.inventory.enabled && settings.inventory.inventoryTools) {
				HijackAngular("assetsExplorer", {
					assetsService(handler, args) {
						const result = handler.apply(this, args)
 
						try {
							const tbuat = result.beginUpdateAssetsItems
							result.beginUpdateAssetsItems = function(...iargs) {
								const promise = tbuat.apply(result, iargs)
 
								ContentJS.send("inventoryUpdateBegin")
								promise.then(() => {
									setTimeout(() => {
										ContentJS.send("inventoryUpdateEnd")
									}, 0)
								})
 
								return promise
							}
						} catch(ex) {
							console.error(ex)
							if(IS_DEV_MODE) { alert("HijackAngular Error") }
						}
 
						return result
					}
				})
			}
 
			if(currentPage === "profile" && settings.profile.enabled) {
				HijackAngular("peopleList", {
					layoutService(handler, args) {
						const result = handler.apply(this, args)
						result.maxNumberOfFriendsDisplayed = 10
						return result
					}
				})
			}
 
			if(currentPage === "messages") {
				HijackAngular("messages", {
					messagesNav(handler, args, argMap) {
						const result = handler.apply(this, args)
 
						try {
							const { $location } = argMap
 
							const link = result.link
							result.link = function(u) {
								try {
									u.btr_setPage = function($event) {
										if($event.which === 13) {
											const value = $event.target.value
 
											if(!Number.isNaN(value)) {
												$location.search({ page: value })
											} else {
												$event.target.value = u.currentStatus.currentPage
											}
 
											$event.preventDefault()
										}
									}
								} catch(ex) {
									console.error(ex)
									if(IS_DEV_MODE) { alert("HijackAngular Error") }
								}
 
								return link.call(this, u)
							}
						} catch(ex) {
							console.error(ex)
							if(IS_DEV_MODE) { alert("HijackAngular Error") }
						}
 
						return result
					}
				})
			}
 
			if(currentPage === "groups" && settings.groups.redesign) {
				if(settings.groups.modifySmallSocialLinksTitle) {
					HijackAngular("socialLinksJumbotron", {
						socialLinkIcon(component) {
							component.bindings.title = "<"
						}
					})
				}
 
				if(settings.groups.pagedGroupWall) {
					const createCustomPager = ({ $scope }) => {
						const wallPosts = []
						const pageSize = 10
						let loadMorePromise = null
						let nextPageCursor = ""
						let requestCounter = 0
						let lastPageNum = 0
						let isLoadingPosts = false
 
						const btrPagerStatus = {
							prev: false,
							next: false,
							input: false,
							pageNum: 1
						}
 
						const setPageNumber = page => {
							btrPagerStatus.prev = page > 0
							btrPagerStatus.next = !!nextPageCursor || wallPosts.length > ((page + 1) * pageSize)
							btrPagerStatus.input = true
							btrPagerStatus.pageNum = page + 1
 
							lastPageNum = page
 
							const startIndex = page * pageSize
							const endIndex = startIndex + pageSize
 
							$scope.groupWall.posts = wallPosts.slice(startIndex, endIndex).map($scope.convertResultToPostObject)
							$scope.$applyAsync()
						}
 
						const loadMorePosts = () => {
							if(loadMorePromise) {
								return loadMorePromise
							}
 
							return loadMorePromise = new Promise(async resolve => {
								const groupId = $scope.library.currentGroup.id
								const baseUrl = `https://groups.roblox.com/v2/groups/${groupId}/wall/posts?sortOrder=Desc&limit=100&cursor=`
 
								const resp = await fetch(baseUrl + nextPageCursor, { credentials: "include" })
								const json = await resp.json()
 
								if(!loadMorePromise) { return }
 
								nextPageCursor = json.nextPageCursor || null
								wallPosts.push(...json.data.filter(x => x.poster))
 
								loadMorePromise = null
								resolve()
							})
						}
 
						const requestWallPosts = page => {
							if(!Number.isSafeInteger(page)) { return }
							const myCounter = ++requestCounter
 
							btrPagerStatus.prev = false
							btrPagerStatus.next = false
							btrPagerStatus.input = false
 
							page = Math.max(0, Math.floor(page))
							isLoadingPosts = true
 
							const startIndex = page * pageSize
							const endIndex = startIndex + pageSize
 
							const tryAgain = () => {
								if(requestCounter !== myCounter) { return }
 
								if(wallPosts.length < endIndex && nextPageCursor !== null) {
									loadMorePosts().then(tryAgain)
									return
								}
 
								const maxPage = Math.max(0, Math.floor((wallPosts.length - 1) / pageSize))
								setPageNumber(Math.min(maxPage, page))
 
								isLoadingPosts = false
							}
 
							tryAgain()
						}
 
						$scope.groupWall.pager.isBusy = () => isLoadingPosts
						$scope.groupWall.pager.loadNextPage = () => {}
						$scope.groupWall.pager.loadFirstPage = () => {
							wallPosts.splice(0, wallPosts.length)
							nextPageCursor = ""
							loadMorePromise = null
							requestWallPosts(0)
						}
 
						$scope.btrPagerStatus = btrPagerStatus
						$scope.btrLoadWallPosts = cursor => {
							let pageNum = lastPageNum
 
							if(cursor === "prev") {
								pageNum = lastPageNum - 1
							} else if(cursor === "next") {
								pageNum = lastPageNum + 1
							} else if(cursor === "input") {
								const input = document.querySelector(".btr-comment-pager input")
								const value = parseInt(input.value, 10)
 
								if(Number.isSafeInteger(value)) {
									pageNum = Math.max(0, value - 1)
								}
							} else if(cursor === "first") {
								pageNum = lastPageNum - 50
							} else if(cursor === "last") {
								pageNum = lastPageNum + 50
							}
 
							requestWallPosts(pageNum)
						}
					}
 
					HijackAngular("group", {
						groupWallController(func, args, argMap) {
							const result = func.apply(this, args)
 
							try {
								createCustomPager(argMap)
							} catch(ex) {
								console.error(ex)
								if(IS_DEV_MODE) { alert("HijackAngular Error") }
							}
 
							return result
						}
					})
				}
			}
		} else {
			console.warn("[BTR] window.angular not set")
		}
 
		if(settings.general.fixAudioVolume) {
			$(document).on("jPlayer_ready", "#MediaPlayerSingleton", ev => {
				const audio = ev.currentTarget.querySelector("audio")
				if(audio) {
					audio.volume = 0.3
				}
			})
		}
 
		if(settings.general.fixAudioPreview) {
			const fixing = {}
 
			ContentJS.listen("fixAudioPreview", (url, blobUrl) => {
				if(!fixing[url]) { return }
				delete fixing[url]
 
				console.warn("[BTRoblox] Fixed broken audio previewer")
 
				document.querySelectorAll(`.MediaPlayerIcon[data-mediathumb-url="${url}"]`).forEach(btn => {
					btn.classList.add("btr-audioFix")
					setTimeout(() => btn.classList.remove("btr-audioFix"), 5e3)
 
					if(btn.classList.contains("icon-pause")) { btn.click() }
 
					btn.dataset.mediathumbUrl = blobUrl
					btn.click()
				})
			})
 
			$(document).on("jPlayer_canplay", "#MediaPlayerSingleton", ev => {
				delete fixing[ev.jPlayer.status.src]
			})
 
			$(document).on("jPlayer_error", "#MediaPlayerSingleton", ev => {
				const errorInfo = ev.jPlayer.error
				const url = errorInfo.context
				const data = fixing[url]
 
				if(errorInfo.type === "e_url" && data) {
					clearTimeout(data.timeout)
					ContentJS.send("fixAudioPreview", url)
				}
			})
 
			$(document).on("jPlayer_loadstart", "#MediaPlayerSingleton", ev => {
				const url = ev.jPlayer.status.src
 
				if(url.includes("rbxcdn.com") && !fixing[url]) {
					const data = fixing[url] = {}
 
					data.timeout = setTimeout(() => {
						if(fixing[url]) {
							ContentJS.send("fixAudioPreview", url)
						}
					}, 500)
				}
			})
		}
 
		if(typeof Roblox !== "undefined") {
			if(settings.general.hideAds) {
				if(Roblox.PrerollPlayer) {
					Roblox.PrerollPlayer.waitForPreroll = x => $.Deferred().resolve(x)
				}
 
				if(Roblox.VideoPreRollDFP) {
					Roblox.VideoPreRollDFP = null
				}
			}
 
			if(currentPage === "gamedetails" && settings.gamedetails.enabled) {
				const placeId = matches[0]
 
				// Server pagers
				const createPager = gameInstance => {
					let curPage = 1
					let maxPage = 1
 
					$(".rbx-running-games-load-more").hide() // Hide Load More
					$(".rbx-running-games-footer > .pager").hide() // Hide Roblox+ pager?
 
					const pager = $(`
					<div class=btr-pager-holder>
						<ul class="pager btr-server-pager">
							<li class=first><a><span class=icon-first-page></a></li>
							<li class=pager-prev><a><span class=icon-left></a></li>
							<li class=pager-mid>
								Page <input class=pager-cur type=text></input>
								of <span class=pager-total></span>
							</li>
							<li class=pager-next><a><span class=icon-right></a></li>
							<li class=last><a><span class=icon-last-page></a></li>
						</ul>
					</div>`).appendTo($(".rbx-running-games-footer"))
 
					const updatePager = () => {
						pager.find(".pager-cur").val(curPage)
						pager.find(".pager-total").text(maxPage)
 
						pager.find(".first").toggleClass("disabled", curPage <= 1)
						pager.find(".pager-prev").toggleClass("disabled", curPage <= 1)
						pager.find(".last").toggleClass("disabled", curPage >= maxPage)
						pager.find(".pager-next").toggleClass("disabled", curPage >= maxPage)
 
						$(".rbx-game-server-join").removeAttr("href")
					}
 
					$.ajaxPrefilter(options => {
						if(!options.url.includes("/games/getgameinstancesjson")) { return }
 
						const startIndex = +new URLSearchParams(options.data).get("startIndex")
						if(!Number.isSafeInteger(startIndex)) { return }
 
						const success = options.success
						options.success = function(...args) {
							curPage = Math.floor(startIndex / 10) + 1
							maxPage = Math.max(1, Math.ceil(args[0].TotalCollectionSize / 10))
 
							$("#rbx-game-server-item-container").find(">.rbx-game-server-item, >.section-content-off").remove()
							updatePager()
 
							if(!args[0].Collection.length) {
								$("#rbx-game-server-item-container").append(`<p class=section-content-off>No Servers Found.</p>`)
							}
 
							return success.apply(this, args)
						}
					})
 
					pager
						.on("click", ".pager-prev:not(.disabled)", () => {
							gameInstance.fetchServers(placeId, Math.max((curPage - 2) * 10, 0))
						})
						.on("click", ".pager-next:not(.disabled)", () => {
							gameInstance.fetchServers(placeId, Math.min(curPage * 10, (maxPage - 1) * 10))
						})
						.on("click", ".first:not(.disabled)", () => {
							gameInstance.fetchServers(placeId, 0)
						})
						.on("click", ".last:not(.disabled)", () => {
							gameInstance.fetchServers(placeId, (maxPage - 1) * 10)
						})
						.on({
							blur() {
								const text = $(this).val()
								let num = parseInt(text, 10)
 
								if(!Number.isNaN(num)) {
									num = Math.max(1, Math.min(maxPage, num))
									gameInstance.fetchServers(placeId, (num - 1) * 10)
								}
							},
							keypress(e) {
								if(e.which === 13) {
									$(this).blur()
								}
							}
						}, ".pager-cur")
				}
 
				const init = () => {
					if(settings.gamedetails.addServerPager) {
						createPager(Roblox.RunningGameInstances)
					}
 
					// Init tab
					const tabBtn = document.querySelector(".rbx-tab.active a")
					if(tabBtn) {
						jQuery(tabBtn).trigger("shown.bs.tab")
					}
				}
 
				if(Roblox.RunningGameInstances) {
					setTimeout(init, 0)
				}
			} else if(currentPage === "develop") {
				if(Roblox.BuildPage) {
					Roblox.BuildPage.GameShowcase = new Proxy(Roblox.BuildPage.GameShowcase || {}, {
						set(target, name, value) {
							target[name] = value
							const table = document.querySelector(`.item-table[data-rootplace-id="${name}"]`)
							if(table) { table.dataset.inShowcase = value }
							return true
						}
					})
				}
			}
		} else {
			if(IS_DEV_MODE) {
				alert("[BTR] window.Roblox not set")
			}
		}
 
		if(typeof Sys !== "undefined" && Sys.WebForms != null) {
			const prm = Sys.WebForms.PageRequestManager.getInstance()
 
			prm.add_pageLoaded(() => ContentJS.send("ajaxUpdate"))
		}
	}
 
	ContentJS.listen("TEMPLATE_INIT", key => templates[key] = true)
	ContentJS.listen("linkify", cl => {
		const target = $(`.${cl}`)
		target.removeClass(cl)
		if(window.Roblox && Roblox.Linkify) { target.linkify() }
		else { target.addClass("linkify") }
	})
 
	ContentJS.listen("INIT", (...initData) => {
		[settings, currentPage, matches, IS_DEV_MODE] = initData
		settingsAreLoaded = true
 
		PostInit()
 
		if(document.readyState === "loading") {
			document.addEventListener("DOMContentLoaded", DocumentReady)
		} else {
			DocumentReady()
		}
	})
 
	PreInit()
})();</script><style name="BTRoblox/inject.css" type="text/css"></style><!--<![endif]--><style type="text/css">[uib-tooltip-popup].tooltip.top-left > .tooltip-arrow,[uib-tooltip-popup].tooltip.top-right > .tooltip-arrow,[uib-tooltip-popup].tooltip.bottom-left > .tooltip-arrow,[uib-tooltip-popup].tooltip.bottom-right > .tooltip-arrow,[uib-tooltip-popup].tooltip.left-top > .tooltip-arrow,[uib-tooltip-popup].tooltip.left-bottom > .tooltip-arrow,[uib-tooltip-popup].tooltip.right-top > .tooltip-arrow,[uib-tooltip-popup].tooltip.right-bottom > .tooltip-arrow,[uib-tooltip-html-popup].tooltip.top-left > .tooltip-arrow,[uib-tooltip-html-popup].tooltip.top-right > .tooltip-arrow,[uib-tooltip-html-popup].tooltip.bottom-left > .tooltip-arrow,[uib-tooltip-html-popup].tooltip.bottom-right > .tooltip-arrow,[uib-tooltip-html-popup].tooltip.left-top > .tooltip-arrow,[uib-tooltip-html-popup].tooltip.left-bottom > .tooltip-arrow,[uib-tooltip-html-popup].tooltip.right-top > .tooltip-arrow,[uib-tooltip-html-popup].tooltip.right-bottom > .tooltip-arrow,[uib-tooltip-template-popup].tooltip.top-left > .tooltip-arrow,[uib-tooltip-template-popup].tooltip.top-right > .tooltip-arrow,[uib-tooltip-template-popup].tooltip.bottom-left > .tooltip-arrow,[uib-tooltip-template-popup].tooltip.bottom-right > .tooltip-arrow,[uib-tooltip-template-popup].tooltip.left-top > .tooltip-arrow,[uib-tooltip-template-popup].tooltip.left-bottom > .tooltip-arrow,[uib-tooltip-template-popup].tooltip.right-top > .tooltip-arrow,[uib-tooltip-template-popup].tooltip.right-bottom > .tooltip-arrow,[uib-popover-popup].popover.top-left > .arrow,[uib-popover-popup].popover.top-right > .arrow,[uib-popover-popup].popover.bottom-left > .arrow,[uib-popover-popup].popover.bottom-right > .arrow,[uib-popover-popup].popover.left-top > .arrow,[uib-popover-popup].popover.left-bottom > .arrow,[uib-popover-popup].popover.right-top > .arrow,[uib-popover-popup].popover.right-bottom > .arrow,[uib-popover-html-popup].popover.top-left > .arrow,[uib-popover-html-popup].popover.top-right > .arrow,[uib-popover-html-popup].popover.bottom-left > .arrow,[uib-popover-html-popup].popover.bottom-right > .arrow,[uib-popover-html-popup].popover.left-top > .arrow,[uib-popover-html-popup].popover.left-bottom > .arrow,[uib-popover-html-popup].popover.right-top > .arrow,[uib-popover-html-popup].popover.right-bottom > .arrow,[uib-popover-template-popup].popover.top-left > .arrow,[uib-popover-template-popup].popover.top-right > .arrow,[uib-popover-template-popup].popover.bottom-left > .arrow,[uib-popover-template-popup].popover.bottom-right > .arrow,[uib-popover-template-popup].popover.left-top > .arrow,[uib-popover-template-popup].popover.left-bottom > .arrow,[uib-popover-template-popup].popover.right-top > .arrow,[uib-popover-template-popup].popover.right-bottom > .arrow{top:auto;bottom:auto;left:auto;right:auto;margin:0;}[uib-popover-popup].popover,[uib-popover-html-popup].popover,[uib-popover-template-popup].popover{display:block !important;}</style><style type="text/css">.uib-position-measure{display:block !important;visibility:hidden !important;position:absolute !important;top:-9999px !important;left:-9999px !important;}.uib-position-scrollbar-measure{position:absolute !important;top:-9999px !important;width:50px !important;height:50px !important;overflow:scroll !important;}.uib-position-body-scrollbar-measure{overflow:scroll !important;}</style><style type="text/css">.ng-animate.item:not(.left):not(.right){-webkit-transition:0s ease-in-out left;transition:0s ease-in-out left}</style><style type="text/css">@charset "UTF-8";[ng\:cloak],[ng-cloak],[data-ng-cloak],[x-ng-cloak],.ng-cloak,.x-ng-cloak,.ng-hide:not(.ng-hide-animate){display:none !important;}ng\:form{display:block;}.ng-animate-shim{visibility:hidden;}.ng-anchor{position:absolute;}</style><title>(1) Home - Roblox</title><meta http-equiv="X-UA-Compatible" content="IE=edge,requiresActiveX=true"><meta charset="UTF-8"><meta name="viewport" content="width=device-width, initial-scale=1"><meta name="author" content="Roblox Corporation"><meta name="description" content="Roblox is a global platform that brings people together through play."><meta name="keywords" content="free games,online games,building games,virtual worlds,free mmo,gaming cloud,physics engine"><meta name="apple-itunes-app" content="app-id=431946152"><script type="application/ld+json">
    {
    "@context" : "http://schema.org",
    "@type" : "Organization",
    "name" : "Roblox",
    "url" : "https://www.roblox.com/",
    "logo": "https://images.rbxcdn.com/c69b74f49e785df33b732273fad9dbe0.png",
    "sameAs" : [
    "https://www.facebook.com/ROBLOX/",
    "https://twitter.com/roblox",
    "https://www.linkedin.com/company/147977",
    "https://www.instagram.com/roblox/",
    "https://www.youtube.com/user/roblox",
    "https://plus.google.com/+roblox",
    "https://www.twitch.tv/roblox"
    ]
    }
</script><meta name="user-data" data-userid="1253745616" data-name="lastquest5847" data-isunder13="false"><meta name="locale-data" data-language-code="en_us" data-language-name="English"><meta name="device-meta" data-device-type="computer" data-is-in-app="false" data-is-desktop="true" data-is-phone="false" data-is-tablet="false" data-is-console="false" data-is-android-app="false" data-is-ios-app="false" data-is-uwp-app="false" data-is-xbox-app="false" data-is-amazon-app="false" data-is-win32-app="false" data-is-studio="false" data-is-game-client-browser="false" data-is-ios-device="false" data-is-android-device="false" data-app-type="unknown"><meta name="page-meta" data-internal-page-name="Home"><meta name="performance" data-ui-performance-relative-value="1" data-ui-performance-endpoint="https://metrics.roblox.com/v1/performance/send-measurement"><script>var Roblox=Roblox||{};Roblox.BundleVerifierConstants={isMetricsApiEnabled:true,eventStreamUrl:"//ecsv2.roblox.com/pe?t=diagnostic",deviceType:"Computer",cdnLoggingEnabled:JSON.parse("true")};</script><script>var Roblox=Roblox||{};Roblox.BundleDetector=(function(){var isMetricsApiEnabled=Roblox.BundleVerifierConstants&&Roblox.BundleVerifierConstants.isMetricsApiEnabled;var loadStates={loadSuccess:"loadSuccess",loadFailure:"loadFailure",executionFailure:"executionFailure"};var bundleContentTypes={javascript:"javascript",css:"css"};var ephemeralCounterNames={cdnPrefix:"CDNBundleError_",unknown:"CDNBundleError_unknown",cssError:"CssBundleError",jsError:"JavascriptBundleError",jsFileError:"JsFileExecutionError",resourceError:"ResourcePerformance_Error",resourceLoaded:"ResourcePerformance_Loaded"};return{jsBundlesLoaded:{},bundlesReported:{},counterNames:ephemeralCounterNames,loadStates:loadStates,bundleContentTypes:bundleContentTypes,timing:undefined,setTiming:function(windowTiming){this.timing=windowTiming;},getLoadTime:function(){if(this.timing&&this.timing.domComplete){return this.getCurrentTime()-this.timing.domComplete;}},getCurrentTime:function(){return new Date().getTime();},getCdnProviderName:function(bundleUrl,callBack){if(Roblox.BundleVerifierConstants.cdnLoggingEnabled){var xhr=new XMLHttpRequest();xhr.open('GET',bundleUrl,true);xhr.onreadystatechange=function(){if(xhr.readyState===xhr.HEADERS_RECEIVED){try{var headerValue=xhr.getResponseHeader("rbx-cdn-provider");if(headerValue){callBack(headerValue);}else{callBack();}}catch(e){callBack();}}};xhr.onerror=function(){callBack();};xhr.send();}else{callBack();}},getCdnProviderAndReportMetrics:function(bundleUrl,bundleName,loadState,bundleContentType){this.getCdnProviderName(bundleUrl,function(cdnProviderName){Roblox.BundleDetector.reportMetrics(bundleUrl,bundleName,loadState,bundleContentType,cdnProviderName);});},reportMetrics:function(bundleUrl,bundleName,loadState,bundleContentType,cdnProviderName){if(!isMetricsApiEnabled||!bundleUrl||!loadState||!loadStates.hasOwnProperty(loadState)||!bundleContentType||!bundleContentTypes.hasOwnProperty(bundleContentType)){return;}
var xhr=new XMLHttpRequest();var metricsApiUrl=(Roblox.EnvironmentUrls&&Roblox.EnvironmentUrls.metricsApi)||"https://metrics.roblox.com";xhr.open("POST",metricsApiUrl+"/v1/bundle-metrics/report",true);xhr.setRequestHeader("Content-Type","application/json");xhr.withCredentials=true;xhr.send(JSON.stringify({bundleUrl:bundleUrl,bundleName:bundleName||"",bundleContentType:bundleContentType,loadState:loadState,cdnProviderName:cdnProviderName,loadTimeInMilliseconds:this.getLoadTime()||0}));},logToEphemeralStatistics:function(sequenceName,value){var deviceType=Roblox.BundleVerifierConstants.deviceType;sequenceName+="_"+deviceType;var xhr=new XMLHttpRequest();xhr.open('POST','/game/report-stats?name='+sequenceName+"&value="+value,true);xhr.withCredentials=true;xhr.send();},logToEphemeralCounter:function(ephemeralCounterName){var deviceType=Roblox.BundleVerifierConstants.deviceType;ephemeralCounterName+="_"+deviceType;var xhr=new XMLHttpRequest();xhr.open('POST','/game/report-event?name='+ephemeralCounterName,true);xhr.withCredentials=true;xhr.send();},logToEventStream:function(failedBundle,ctx,cdnProvider,status){var esUrl=Roblox.BundleVerifierConstants.eventStreamUrl,currentPageUrl=encodeURIComponent(window.location.href);var deviceType=Roblox.BundleVerifierConstants.deviceType;ctx+="_"+deviceType;var duration=0;if(window.performance){var perfTiming=window.performance.getEntriesByName(failedBundle);if(perfTiming.length>0){var data=perfTiming[0];duration=data.duration||0;}}
var params="&evt=webBundleError&url="+currentPageUrl+"&ctx="+ctx+"&fileSourceUrl="+encodeURIComponent(failedBundle)+"&cdnName="+(cdnProvider||"unknown")+"&statusCode="+(status||"unknown")+"&loadDuration="+Math.floor(duration);var img=new Image();img.src=esUrl+params;},getCdnInfo:function(failedBundle,ctx,fileType){if(Roblox.BundleVerifierConstants.cdnLoggingEnabled){var xhr=new XMLHttpRequest();var counter=this.counterNames;xhr.open('GET',failedBundle,true);var cdnProvider;xhr.onreadystatechange=function(){if(xhr.readyState===xhr.HEADERS_RECEIVED){cdnProvider=xhr.getResponseHeader("rbx-cdn-provider");if(cdnProvider&&cdnProvider.length>0){Roblox.BundleDetector.logToEphemeralCounter(counter.cdnPrefix+cdnProvider+"_"+fileType);}
else{Roblox.BundleDetector.logToEphemeralCounter(counter.unknown+"_"+fileType);}}
else if(xhr.readyState===xhr.DONE){Roblox.BundleDetector.logToEventStream(failedBundle,ctx,cdnProvider,xhr.status);}};xhr.onerror=function(){Roblox.BundleDetector.logToEphemeralCounter(counter.unknown+"_"+fileType);Roblox.BundleDetector.logToEventStream(failedBundle,ctx,counter.unknown);};xhr.send();}
else{this.logToEventStream(failedBundle,ctx);}},reportResourceError:function(resourceName){var ephemeralCounterName=this.counterNames.resourceError+"_"+resourceName;this.logToEphemeralCounter(ephemeralCounterName);},reportResourceLoaded:function(resourceName){var loadTimeInMs=this.getLoadTime();if(loadTimeInMs){var sequenceName=this.counterNames.resourceLoaded+"_"+resourceName;this.logToEphemeralStatistics(sequenceName,loadTimeInMs);}},reportBundleError:function(bundleTag){var ephemeralCounterName,failedBundle,ctx,contentType;if(bundleTag.rel&&bundleTag.rel==="stylesheet"){ephemeralCounterName=this.counterNames.cssError;failedBundle=bundleTag.href;ctx="css";contentType=bundleContentTypes.css;}else{ephemeralCounterName=this.counterNames.jsError;failedBundle=bundleTag.src;ctx="js";contentType=bundleContentTypes.javascript;}
this.bundlesReported[failedBundle]=true;this.logToEphemeralCounter(ephemeralCounterName);this.getCdnInfo(failedBundle,ctx,ctx);var bundleName;if(bundleTag.dataset){bundleName=bundleTag.dataset.bundlename;}
else{bundleName=bundleTag.getAttribute('data-bundlename');}
this.getCdnProviderAndReportMetrics(failedBundle,bundleName,loadStates.loadFailure,contentType);},bundleDetected:function(bundleName){this.jsBundlesLoaded[bundleName]=true;},verifyBundles:function(document){var ephemeralCounterName=this.counterNames.jsFileError,eventContext=ephemeralCounterName;var scripts=(document&&document.scripts)||window.document.scripts;var errorsList=[];var bundleName;var monitor;for(var i=0;i<scripts.length;i++){var item=scripts[i];if(item.dataset){bundleName=item.dataset.bundlename;monitor=item.dataset.monitor;}
else{bundleName=item.getAttribute('data-bundlename');monitor=item.getAttribute('data-monitor');}
if(item.src&&monitor&&bundleName){if(!Roblox.BundleDetector.jsBundlesLoaded.hasOwnProperty(bundleName)){errorsList.push(item);}}}
if(errorsList.length>0){for(var j=0;j<errorsList.length;j++){var script=errorsList[j];if(!this.bundlesReported[script.src]){this.logToEphemeralCounter(ephemeralCounterName);this.getCdnInfo(script.src,eventContext,'js');if(script.dataset){bundleName=script.dataset.bundlename;}
else{bundleName=script.getAttribute('data-bundlename');}
this.getCdnProviderAndReportMetrics(script.src,bundleName,loadStates.executionFailure,bundleContentTypes.javascript);}}}}};})();window.addEventListener("load",function(evt){Roblox.BundleDetector.verifyBundles();});Roblox.BundleDetector.setTiming(window.performance.timing);</script><link href="https://images.rbxcdn.com/23421382939a9f4ae8bbe60dbe2a3e7e.ico.gzip" rel="icon"><link rel="manifest" href="https://notifications.roblox.com/v2/push-notifications/chrome-manifest" crossorigin="use-credentials"><link onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" rel="stylesheet" data-bundlename="Thumbnails" href="https://static.rbxcdn.com/css/72cd3aca154fd66b2ada809c31d17a2ee0cf653f89ccbbffe4e44025a4afd35e.css/fetch"><link onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" rel="stylesheet" data-bundlename="StyleGuide" href="https://static.rbxcdn.com/css/b5bb43dc638fec383967b9213abd937583decaf992ca9f8a5c089dc7ac8d04eb.css/fetch"><link onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" rel="stylesheet" data-bundlename="Footer" href="https://static.rbxcdn.com/css/55b250e8473888792f885d898973a13692fb22157baf61aaffa62ce4545f3408.css/fetch"><link rel="canonical" href="https://www.roblox.com/home"><link onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" rel="stylesheet" href="https://static.rbxcdn.com/css/leanbase___3678d89e5ec3f4d8c65d863691f31de2_m.css/fetch"><link onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" rel="stylesheet" href="https://static.rbxcdn.com/css/page___03aa53ed13d56a69d25f9ef7eb102fa3_m.css/fetch"><link onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" rel="stylesheet" data-bundlename="PeopleList" href="https://static.rbxcdn.com/css/d708fd4aace739f827de423a092019fb52f893d6d36c80cfbcdd98c040c74e75.css/fetch"><link onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" rel="stylesheet" data-bundlename="PlacesList" href="https://static.rbxcdn.com/css/3e5a7ee1c6d65f8a37ed7bcb50c65e399df74ac92d177bc94f75679b056a39b2.css/fetch"><link onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" rel="stylesheet" data-bundlename="RobuxIcon" href="https://static.rbxcdn.com/css/af4a705d9238d48149768cbd4724797649ca06ff6dbf0b05feab30c7825997be.css/fetch"><link onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" rel="stylesheet" data-bundlename="NotificationStream" href="https://static.rbxcdn.com/css/c5eab44ee3b34acdae36b6dad3297240134fbaaba8f2a77634bf0f893eeafabd.css/fetch"><link onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" rel="stylesheet" data-bundlename="Chat" href="https://static.rbxcdn.com/css/9363da0bd9d79e2636af8699417f40c5884029ed0e48e049edaa1957afde8901.css/fetch"><script>var Roblox=Roblox||{};Roblox.RealTimeSettings=Roblox.RealTimeSettings||{NotificationsEndpoint:"https://realtime.roblox.com",MaxConnectionTime:"21600000",IsEventPublishingEnabled:false,IsDisconnectOnSlowConnectionDisabled:true,IsSignalRClientTransportRestrictionEnabled:true,IsLocalStorageInRealTimeEnabled:true,IsDebuggerEnabled:"False"}</script><script>var Roblox=Roblox||{};Roblox.EnvironmentUrls=Roblox.EnvironmentUrls||{};Roblox.EnvironmentUrls={"abtestingApiSite":"https://abtesting.roblox.com","accountInformationApi":"https://accountinformation.roblox.com","accountSettingsApi":"https://accountsettings.roblox.com","adsApi":"https://ads.roblox.com","apiGatewayUrl":"https://apis.roblox.com","apiProxyUrl":"https://api.roblox.com","assetDeliveryApi":"https://assetdelivery.roblox.com","authApi":"https://auth.roblox.com","authAppSite":"https://authsite.roblox.com","avatarApi":"https://avatar.roblox.com","badgesApi":"https://badges.roblox.com","billingApi":"https://billing.roblox.com","captchaApi":"https://captcha.roblox.com","catalogApi":"https://catalog.roblox.com","chatApi":"https://chat.roblox.com","contactsApi":"https://contacts.roblox.com","developApi":"https://develop.roblox.com","domain":"roblox.com","economyApi":"https://economy.roblox.com","engagementPayoutsApi":"https://engagementpayouts.roblox.com","followingsApi":"https://followings.roblox.com","friendsApi":"https://friends.roblox.com","friendsAppSite":"https://friendsite.roblox.com","gamesApi":"https://games.roblox.com","gameInternationalizationApi":"https://gameinternationalization.roblox.com","groupsApi":"https://groups.roblox.com","inventoryApi":"https://inventory.roblox.com","itemConfigurationApi":"https://itemconfiguration.roblox.com","localeApi":"https://locale.roblox.com","localizationTablesApi":"https://localizationtables.roblox.com","metricsApi":"https://metrics.roblox.com","midasApi":"https://midas.roblox.com","notificationApi":"https://notifications.roblox.com","premiumFeaturesApi":"https://premiumfeatures.roblox.com","presenceApi":"https://presence.roblox.com","publishApi":"https://publish.roblox.com","screenTimeApi":"https://apis.rcs.roblox.com/screen-time-api","thumbnailsApi":"https://thumbnails.roblox.com","tradesApi":"https://trades.roblox.com","translationRolesApi":"https://translationroles.roblox.com","universalAppConfigurationApi":"https://apis.roblox.com/universal-app-configuration","usersApi":"https://users.roblox.com","voiceApi":"https://voice.roblox.com","websiteUrl":"https://www.roblox.com","privateMessagesApi":"https://privatemessages.roblox.com"};var additionalUrls={amazonStoreLink:"https://www.amazon.com/Roblox-Corporation/dp/B00NUF4YOA",appProtocolUrl:"robloxmobile://",appStoreLink:"https://itunes.apple.com/us/app/roblox-mobile/id431946152",googlePlayStoreLink:"https://play.google.com/store/apps/details?id=com.roblox.client&amp;hl=en",iosAppStoreLink:"https://itunes.apple.com/us/app/roblox-mobile/id431946152",windowsStoreLink:"https://www.microsoft.com/en-us/store/games/roblox/9nblgggzm6wm",xboxStoreLink:"https://www.microsoft.com/en-us/p/roblox/bq1tn1t79v9k",amazonWebStoreLink:"https://www.amazon.com/roblox?&amp;_encoding=UTF8&amp;tag=r05d13-20&amp;linkCode=ur2&amp;linkId=4ba2e1ad82f781c8e8cc98329b1066d0&amp;camp=1789&amp;creative=9325"}
for(var urlName in additionalUrls){Roblox.EnvironmentUrls[urlName]=additionalUrls[urlName];}</script><script>var Roblox=Roblox||{};Roblox.GaEventSettings={gaDFPPreRollEnabled:"false"==="true",gaLaunchAttemptAndLaunchSuccessEnabled:"false"==="true",gaPerformanceEventEnabled:"false"==="true"};</script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="headerinit" src="https://js.rbxcdn.com/799efe9bfd5be7618e023fc94f1b1b84.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="RealTime" src="https://js.rbxcdn.com/4cb4fa56ba675608e2cbd2f0bc7bfa932969af63bd7a87ef73cd23558b7c39e4.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="CrossTabCommunication" src="https://js.rbxcdn.com/6f451b71ad4e130aa7f8a1a91b8b6a0974f1237d4f830b8a642ad2c8f5cc05d4.js"></script><meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=0"><script>var Roblox=Roblox||{};Roblox.AdsHelper=Roblox.AdsHelper||{};Roblox.AdsLibrary=Roblox.AdsLibrary||{};Roblox.AdsHelper.toggleAdsSlot=function(slotId,GPTRandomSlotIdentifier){var gutterAdsEnabled=false;if(gutterAdsEnabled){googletag.display(GPTRandomSlotIdentifier);return;}
if(typeof slotId!=='undefined'&&slotId&&slotId.length>0){var slotElm=$("#"+slotId);if(slotElm.is(":visible")){googletag.display(GPTRandomSlotIdentifier);}else{var adParam=Roblox.AdsLibrary.adsParameters[slotId];if(adParam){adParam.template=slotElm.html();slotElm.empty();}}}}</script><script>$(function(){Roblox.JSErrorTracker.initialize({'suppressConsoleError':true});});</script><!--[if lt IE 9]><script src=//oss.maxcdn.com/html5shiv/3.7.2/html5shiv.min.js></script><script src=//oss.maxcdn.com/respond/1.4.2/respond.min.js></script><![endif]--><script>var Roblox=Roblox||{};(function(){var dnt=navigator.doNotTrack||window.doNotTrack||navigator.msDoNotTrack;if(typeof window.external!=="undefined"&&typeof window.external.msTrackingProtectionEnabled!=="undefined"){dnt=dnt||window.external.msTrackingProtectionEnabled();}
Roblox.browserDoNotTrack=dnt=="1"||dnt=="yes"||dnt===true;})();</script><script>if(Roblox&&Roblox.EventStream){Roblox.EventStream.Init("","","","");}</script><script>if(Roblox&&Roblox.PageHeartbeatEvent){Roblox.PageHeartbeatEvent.Init([2,8,20,60]);}</script><script>if(typeof(Roblox)==="undefined"){Roblox={};}
Roblox.Endpoints=Roblox.Endpoints||{};Roblox.Endpoints.Urls=Roblox.Endpoints.Urls||{};Roblox.Endpoints.Urls['/asset/']='https://assetgame.roblox.com/asset/';Roblox.Endpoints.Urls['/client-status/set']='https://www.roblox.com/client-status/set';Roblox.Endpoints.Urls['/client-status']='https://www.roblox.com/client-status';Roblox.Endpoints.Urls['/game/']='https://assetgame.roblox.com/game/';Roblox.Endpoints.Urls['/game/edit.ashx']='https://assetgame.roblox.com/game/edit.ashx';Roblox.Endpoints.Urls['/game/placelauncher.ashx']='https://assetgame.roblox.com/game/placelauncher.ashx';Roblox.Endpoints.Urls['/game/preloader']='https://assetgame.roblox.com/game/preloader';Roblox.Endpoints.Urls['/game/report-stats']='https://assetgame.roblox.com/game/report-stats';Roblox.Endpoints.Urls['/game/report-event']='https://assetgame.roblox.com/game/report-event';Roblox.Endpoints.Urls['/game/updateprerollcount']='https://assetgame.roblox.com/game/updateprerollcount';Roblox.Endpoints.Urls['/login/default.aspx']='https://www.roblox.com/login/default.aspx';Roblox.Endpoints.Urls['/my/avatar']='https://www.roblox.com/my/avatar';Roblox.Endpoints.Urls['/my/money.aspx']='https://www.roblox.com/my/money.aspx';Roblox.Endpoints.Urls['/navigation/userdata']='https://www.roblox.com/navigation/userdata';Roblox.Endpoints.Urls['/chat/chat']='https://www.roblox.com/chat/chat';Roblox.Endpoints.Urls['/chat/data']='https://www.roblox.com/chat/data';Roblox.Endpoints.Urls['/presence/users']='https://www.roblox.com/presence/users';Roblox.Endpoints.Urls['/presence/user']='https://www.roblox.com/presence/user';Roblox.Endpoints.Urls['/friends/list']='https://www.roblox.com/friends/list';Roblox.Endpoints.Urls['/navigation/getcount']='https://www.roblox.com/navigation/getCount';Roblox.Endpoints.Urls['/regex/email']='https://www.roblox.com/regex/email';Roblox.Endpoints.Urls['/catalog/browse.aspx']='https://www.roblox.com/catalog/browse.aspx';Roblox.Endpoints.Urls['/catalog/html']='https://search.roblox.com/catalog/html';Roblox.Endpoints.Urls['/catalog/json']='https://search.roblox.com/catalog/json';Roblox.Endpoints.Urls['/catalog/contents']='https://search.roblox.com/catalog/contents';Roblox.Endpoints.Urls['/catalog/lists.aspx']='https://search.roblox.com/catalog/lists.aspx';Roblox.Endpoints.Urls['/catalog/items']='https://search.roblox.com/catalog/items';Roblox.Endpoints.Urls['/asset-hash-thumbnail/image']='https://assetgame.roblox.com/asset-hash-thumbnail/image';Roblox.Endpoints.Urls['/asset-hash-thumbnail/json']='https://assetgame.roblox.com/asset-hash-thumbnail/json';Roblox.Endpoints.Urls['/asset-thumbnail-3d/json']='https://assetgame.roblox.com/asset-thumbnail-3d/json';Roblox.Endpoints.Urls['/asset-thumbnail/image']='https://assetgame.roblox.com/asset-thumbnail/image';Roblox.Endpoints.Urls['/asset-thumbnail/json']='https://assetgame.roblox.com/asset-thumbnail/json';Roblox.Endpoints.Urls['/asset-thumbnail/url']='https://assetgame.roblox.com/asset-thumbnail/url';Roblox.Endpoints.Urls['/asset/request-thumbnail-fix']='https://assetgame.roblox.com/asset/request-thumbnail-fix';Roblox.Endpoints.Urls['/avatar-thumbnail-3d/json']='https://www.roblox.com/avatar-thumbnail-3d/json';Roblox.Endpoints.Urls['/avatar-thumbnail/image']='https://www.roblox.com/avatar-thumbnail/image';Roblox.Endpoints.Urls['/avatar-thumbnail/json']='https://www.roblox.com/avatar-thumbnail/json';Roblox.Endpoints.Urls['/avatar-thumbnails']='https://www.roblox.com/avatar-thumbnails';Roblox.Endpoints.Urls['/avatar/request-thumbnail-fix']='https://www.roblox.com/avatar/request-thumbnail-fix';Roblox.Endpoints.Urls['/bust-thumbnail/json']='https://www.roblox.com/bust-thumbnail/json';Roblox.Endpoints.Urls['/group-thumbnails']='https://www.roblox.com/group-thumbnails';Roblox.Endpoints.Urls['/groups/getprimarygroupinfo.ashx']='https://www.roblox.com/groups/getprimarygroupinfo.ashx';Roblox.Endpoints.Urls['/headshot-thumbnail/json']='https://www.roblox.com/headshot-thumbnail/json';Roblox.Endpoints.Urls['/item-thumbnails']='https://www.roblox.com/item-thumbnails';Roblox.Endpoints.Urls['/outfit-thumbnail/json']='https://www.roblox.com/outfit-thumbnail/json';Roblox.Endpoints.Urls['/place-thumbnails']='https://www.roblox.com/place-thumbnails';Roblox.Endpoints.Urls['/thumbnail/asset/']='https://www.roblox.com/thumbnail/asset/';Roblox.Endpoints.Urls['/thumbnail/avatar-headshot']='https://www.roblox.com/thumbnail/avatar-headshot';Roblox.Endpoints.Urls['/thumbnail/avatar-headshots']='https://www.roblox.com/thumbnail/avatar-headshots';Roblox.Endpoints.Urls['/thumbnail/user-avatar']='https://www.roblox.com/thumbnail/user-avatar';Roblox.Endpoints.Urls['/thumbnail/resolve-hash']='https://www.roblox.com/thumbnail/resolve-hash';Roblox.Endpoints.Urls['/thumbnail/place']='https://www.roblox.com/thumbnail/place';Roblox.Endpoints.Urls['/thumbnail/get-asset-media']='https://www.roblox.com/thumbnail/get-asset-media';Roblox.Endpoints.Urls['/thumbnail/remove-asset-media']='https://www.roblox.com/thumbnail/remove-asset-media';Roblox.Endpoints.Urls['/thumbnail/set-asset-media-sort-order']='https://www.roblox.com/thumbnail/set-asset-media-sort-order';Roblox.Endpoints.Urls['/thumbnail/place-thumbnails']='https://www.roblox.com/thumbnail/place-thumbnails';Roblox.Endpoints.Urls['/thumbnail/place-thumbnails-partial']='https://www.roblox.com/thumbnail/place-thumbnails-partial';Roblox.Endpoints.Urls['/thumbnail_holder/g']='https://www.roblox.com/thumbnail_holder/g';Roblox.Endpoints.Urls['/users/{id}/profile']='https://www.roblox.com/users/{id}/profile';Roblox.Endpoints.Urls['/service-workers/push-notifications']='https://www.roblox.com/service-workers/push-notifications';Roblox.Endpoints.Urls['/notification-stream/notification-stream-data']='https://www.roblox.com/notification-stream/notification-stream-data';Roblox.Endpoints.Urls['/api/friends/acceptfriendrequest']='https://www.roblox.com/api/friends/acceptfriendrequest';Roblox.Endpoints.Urls['/api/friends/declinefriendrequest']='https://www.roblox.com/api/friends/declinefriendrequest';Roblox.Endpoints.Urls['/authentication/is-logged-in']='https://www.roblox.com/authentication/is-logged-in';Roblox.Endpoints.addCrossDomainOptionsToAllRequests=true;</script><script>if(typeof(Roblox)==="undefined"){Roblox={};}
Roblox.Endpoints=Roblox.Endpoints||{};Roblox.Endpoints.Urls=Roblox.Endpoints.Urls||{};</script><script>Roblox=Roblox||{};Roblox.AbuseReportPVMeta={desktopEnabled:true,phoneEnabled:false,inAppEnabled:false};</script><style type="text/css"></style></head><body id="rbx-body" class="rbx-body dark-theme gotham-font btr-no-hamburger btr-hide-ads btr-small-chat-button" data-performance-relative-value="0.005" data-internal-page-name="Home" data-send-event-percentage="0" data-btr-page="home"><div id="roblox-linkify" data-enabled="true" data-regex="(https?:\/\/)?([a-z0-9-]+\.)*(twitter\.com|youtube\.com|youtu\.be|twitch\.tv|roblox\.com|robloxlabs\.com|shoproblox\.com)(?!\/[A-Za-z0-9-+&amp;@#/=~_|!:,.;]*%)((\/[A-Za-z0-9-+&amp;@#/%?=~_|!:,.;]*)|(?=\s|\b))" data-regex-flags="gm" data-as-http-regex="(([^.]help|polls)\.roblox\.com)"></div><div id="image-retry-data" data-image-retry-max-times="10" data-image-retry-timer="1500" data-ga-logging-percent="10"></div><div id="http-retry-data" data-http-retry-max-timeout="0" data-http-retry-base-timeout="0" data-http-retry-max-times="1"></div><div id="TosAgreementInfo" data-terms-check-needed="False"></div><div id="fb-root"></div><div id="wrap" class="wrap no-gutter-ads logged-in" data-gutter-ads-enabled="false"><div id="header" class="navbar-fixed-top rbx-header dark-theme gotham-font" data-isauthenticated="true" role="navigation"><div class="container-fluid btr-custom-header"><div class="btr-header-flex"><div class="rbx-navbar-header"><div data-behavior="nav-notification" class="rbx-nav-collapse" onselectstart="return false"><span class="icon-nav-menu"></span></div><div class="navbar-header"><a class="navbar-brand" href="https://www.roblox.com/"> <span class="icon-logo"></span> <span class="icon-logo-r"></span> </a></div></div><ul class="nav rbx-navbar hidden-xs hidden-sm col-md-5 col-lg-4"><li class="cursor-pointer"><a class="font-header-2 nav-menu-title text-header" href="/home">Home</a></li><li class="cursor-pointer"><a class="font-header-2 nav-menu-title text-header" href="https://www.roblox.com/games">Games</a></li><li class="cursor-pointer"><a class="font-header-2 nav-menu-title text-header" href="https://www.roblox.com/catalog/">Avatar Shop</a></li><li class="cursor-pointer"><a class="font-header-2 nav-menu-title text-header" href="https://www.roblox.com/develop">Create</a></li><li class="cursor-pointer btr-nav-disabled"><a class="font-header-2 buy-robux nav-menu-title text-header" href="https://www.roblox.com/upgrades/robux?ctx=nav">Robux</a></li></ul><div id="navbar-universal-search" class="navbar-left rbx-navbar-search col-xs-5 col-sm-6 col-md-2 col-lg-3" data-behavior="univeral-search" role="search"><div class="input-group"><input id="navbar-search-input" class="form-control input-field" type="text" placeholder="Search" maxlength="120"><div class="input-group-btn"><button id="navbar-search-btn" class="input-addon-btn" type="submit"> <span class="icon-nav-search"></span> </button></div></div><ul data-toggle="dropdown-menu" class="dropdown-menu" role="menu"><li class="rbx-navbar-search-option rbx-clickable-li selected" data-searchurl="https://www.roblox.com/games/?Keyword="><a class="rbx-navbar-search-anchor" href="https://www.roblox.com/games/?Keyword="> <span class="rbx-navbar-search-text"> Search "<span class="rbx-navbar-search-string"></span>" in Games</span> </a></li><li class="rbx-navbar-search-option rbx-clickable-li" data-searchurl="https://www.roblox.com/search/users?keyword="><a class="rbx-navbar-search-anchor" href="https://www.roblox.com/search/users?keyword="> <span class="rbx-navbar-search-text"> Search "<span class="rbx-navbar-search-string"></span>" in Players</span> </a></li><li class="rbx-navbar-search-option rbx-clickable-li" data-searchurl="https://www.roblox.com/catalog/browse.aspx?CatalogContext=1&amp;Keyword="><a class="rbx-navbar-search-anchor" href="https://www.roblox.com/catalog/browse.aspx?CatalogContext=1&amp;Keyword="> <span class="rbx-navbar-search-text"> Search "<span class="rbx-navbar-search-string"></span>" in Catalog</span> </a></li><li class="rbx-navbar-search-option rbx-clickable-li" data-searchurl="https://www.roblox.com/search/groups?keyword="><a class="rbx-navbar-search-anchor" href="https://www.roblox.com/search/groups?keyword="> <span class="rbx-navbar-search-text"> Search "<span class="rbx-navbar-search-string"></span>" in Groups</span> </a></li><li class="rbx-navbar-search-option rbx-clickable-li" data-searchurl="https://www.roblox.com/develop/library?CatalogContext=2&amp;Category=6&amp;Keyword="><a class="rbx-navbar-search-anchor" href="https://www.roblox.com/develop/library?CatalogContext=2&amp;Category=6&amp;Keyword="> <span class="rbx-navbar-search-text"> Search "<span class="rbx-navbar-search-string"></span>" in Library</span> </a></li></ul></div><div class="navbar-right rbx-navbar-right"><ul class="nav navbar-right rbx-navbar-icon-group"><li id="navbar-setting" class="navbar-icon-item"><a class="rbx-menu-item roblox-popover-close" data-toggle="popover" data-bind="popover-setting" data-viewport="#header" data-original-title="" title=""> <span class="icon-nav-settings roblox-popover-close" id="nav-settings"></span> <span class="notification-red notification nav-setting-highlight hidden">0</span> </a><div class="rbx-popover-content" data-toggle="popover-setting"><ul class="dropdown-menu" role="menu"><li><a class="rbx-menu-item btr-settings-toggle">BTR Settings</a></li><li><a class="rbx-menu-item" href="https://www.roblox.com/my/account"> Settings <span class="notification-blue notification nav-setting-highlight hidden">0</span> </a></li><li><a class="rbx-menu-item" href="https://www.roblox.com/info/help?locale=en_us" target="_blank">Help</a></li><li><a class="rbx-menu-item" data-behavior="logout" data-bind="https://auth.roblox.com/v2/logout">Logout</a></li></ul></div></li><li id="navbar-robux" class="navbar-icon-item"><a id="nav-robux-icon" class="nav-robux-icon rbx-menu-item" data-toggle="popover" data-bind="popover-robux" data-original-title="" title=""> <span class="icon-robux-28x28 roblox-popover-close" id="nav-robux"></span> <span class="rbx-text-navbar-right text-header" id="nav-robux-amount">100B+</span> </a><div class="rbx-popover-content" data-toggle="popover-robux"><ul class="dropdown-menu" role="menu"><li><a href="https://www.roblox.com/My/Money.aspx#/#Summary_tab" id="nav-robux-balance" class="rbx-menu-item">100,532,422,231 Robux</a></li><li><a href="/develop/developer-exchange" class="rbx-menu-item">$1,255,398,622.61 USD</a></li><li><a href="https://www.roblox.com/upgrades/robux?ctx=navpopover" class="rbx-menu-item">Buy Robux</a></li></ul></div></li><li id="btr-navbar-messages" class="navbar-icon-item"><a class="rbx-menu-item" href="https://www.roblox.com/my/messages/#!/inbox"><span class="icon-nav-message-btr"></span><span class="btr-nav-notif rbx-text-navbar-right" style="">5</span></a></li><li id="btr-navbar-friends" class="navbar-icon-item"><a class="rbx-menu-item" href="https://www.roblox.com/users/friends"><span class="icon-nav-friend-btr"></span><span class="btr-nav-notif rbx-text-navbar-right" style="display:none;">0</span></a></li><li class="navbar-icon-item navbar-stream"><div class="notification-stream" ng-class="{'inApp': library.inApp}" id="notification-stream-icon-container" notification-stream-icon=""> <a id="nav-ns-icon" class="roblox-popover rbx-menu-item notification-stream-icon" data-bind="notification-stream-base" data-container="notification-stream-container" notification-indicator=""> <span class="icon-nav-notification-stream" id="nav-notifications"></span> <span class="notification-red notification ng-binding" ng-show="layout.unreadNotifications > 0 &amp;&amp; (!layout.isNotificationContentOpen)"> 2 </span> </a> </div></li><li class="rbx-navbar-right-search" data-toggle="toggle-search"><a class="rbx-menu-icon rbx-menu-item"> <span class="icon-nav-search-white"></span> </a></li></ul><div class="xsmall age-bracket-label text-header btr-nav-disabled"><span class="age-bracket-label-username font-caption-header">builderman: </span>13+</div></div></div><ul class="nav rbx-navbar hidden-md hidden-lg col-xs-12"><li class="cursor-pointer"><a class="font-header-2 nav-menu-title text-header" href="/home">Home</a></li><li class="cursor-pointer"><a class="font-header-2 nav-menu-title text-header" href="https://www.roblox.com/games">Games</a></li><li class="cursor-pointer"><a class="font-header-2 nav-menu-title text-header" href="https://www.roblox.com/catalog/">Avatar Shop</a></li><li class="cursor-pointer"><a class="font-header-2 nav-menu-title text-header" href="https://www.roblox.com/develop">Create</a></li><li class="cursor-pointer btr-nav-disabled"><a class="font-header-2 buy-robux nav-menu-title text-header" href="https://www.roblox.com/upgrades/robux?ctx=nav">Robux</a></li></ul></div></div><div id="navigation" class="rbx-left-col dark-theme gotham-font" data-behavior="left-col"><ul><li class="text-lead"><a class="text-nav font-header-2 text-overflow" href="https://www.roblox.com/users/1253745616/profile">builderman</a></li><li class="rbx-divider"></li></ul><div class="rbx-scrollbar mCustomScrollbar _mCS_1 mCS_no_scrollbar" data-toggle="scrollbar" onselectstart="return false"><div id="mCSB_1" class="mCustomScrollBox mCS-light mCSB_vertical mCSB_inside" tabindex="0" style="max-height: 0px;"><div id="mCSB_1_container" class="mCSB_container" style="position:relative; top:0; left:0;" dir="ltr"><ul class="left-col-list"><li><a href="https://www.roblox.com/home" id="nav-home" class="dynamic-overflow-container text-nav"><div><span class="icon-nav-home"></span></div><span class="font-header-2 dynamic-ellipsis-item">Home</span> </a></li><li><a href="https://www.roblox.com/users/1253745616/profile" id="nav-profile" class="dynamic-overflow-container text-nav"><div><span class="icon-nav-profile"></span></div><span class="font-header-2 dynamic-ellipsis-item">Profile</span> </a></li><li id="navigation-messages" class="btr-nav-disabled"><a href="https://www.roblox.com/my/messages/#!/inbox" id="nav-message" data-count="5" class="dynamic-overflow-container text-nav"><div><span class="icon-nav-message"></span></div><span class="font-header-2 dynamic-ellipsis-item" title="Messages">Messages</span><div class="dynamic-width-item align-right"><span class="notification-blue notification" title="5">5</span></div></a></li><li id="navigation-friends" class="btr-nav-disabled"><a href="https://www.roblox.com/users/friends" id="nav-friends" data-count="0" class="dynamic-overflow-container text-nav"><div><span class="icon-nav-friends"></span></div><span class="font-header-2 dynamic-ellipsis-item" title="Friends">Friends</span><div class="dynamic-width-item align-right"><span class="notification-blue notification hide" title="0"></span></div></a></li><li><a href="https://www.roblox.com/my/avatar" id="nav-character" class="dynamic-overflow-container text-nav"><div><span class="icon-nav-charactercustomizer"></span></div><span class="font-header-2 dynamic-width-item">Avatar</span> </a></li><li><a href="https://www.roblox.com/users/1253745616/inventory" id="nav-inventory" class="dynamic-overflow-container text-nav"><div><span class="icon-nav-inventory"></span></div><span class="font-header-2 dynamic-width-item">Inventory</span> </a></li><li class="btr-nav-disabled"><a href="https://www.roblox.com/my/money.aspx#/#TradeItems_tab" id="nav-trade" class="dynamic-overflow-container text-nav" data-count="0"><div><span class="icon-nav-trade"></span></div><span class="font-header-2 dynamic-ellipsis-item">Trade</span><div class="dynamic-width-item align-right"><span class="notification-blue notification hide" title="0"></span></div></a></li><li><a href="https://www.roblox.com/my/money.aspx" class="dynamic-overflow-container text-nav"><div><span class="icon-nav-trade"></span></div><span class="font-header-2 dynamic-ellipsis-item">Money</span></a></li><li><a href="https://www.roblox.com/my/groups" id="nav-group" class="dynamic-overflow-container text-nav"><div><span class="icon-nav-group"></span></div><span class="font-header-2 dynamic-ellipsis-item">Groups</span> </a></li><li><a href="https://www.roblox.com/feeds/" id="nav-my-feed" class="dynamic-overflow-container text-nav"><div><span class="icon-nav-my-feed"></span></div><span class="font-header-2 dynamic-ellipsis-item">My Feed</span> </a></li><li><a href="/premium/membership" id="nav-premium" class="dynamic-overflow-container text-nav"><div><span class="icon-nav-premium-btr"></span></div><span class="font-header-2 dynamic-ellipsis-item">Premium</span></a></li><li><a href="https://blog.roblox.com" id="nav-blog" class="dynamic-overflow-container text-nav"><div><span class="icon-nav-blog"></span></div><span class="font-header-2 dynamic-ellipsis-item">Blog</span> </a></li><li id="btr-blogfeed"><a class="btr-feed" href="https://blog.roblox.com/2020/01/mobile-avatar-editor-gets-makeover/"><div class="btr-feedtitle">The Mobile Avatar Editor Gets a Makeover <span class="btr-feeddate">(1 week ago)</span></div><div class="btr-feeddesc">Even the avatar editor needs a new look. See what’s new on iOS and Android mobile devices!</div></a><a class="btr-feed" href="https://blog.roblox.com/2020/01/2019-year-review/"><div class="btr-feedtitle">2019 Year-in-Review <span class="btr-feeddate">(2 weeks ago)</span></div><div class="btr-feeddesc">A look back at the milestones and highlights of 2019 with our founder and CEO, David Baszucki.</div></a><a class="btr-feed" href="https://blog.roblox.com/2019/12/help-kids-teens-get-holiday-game-time/"><div class="btr-feedtitle">Help Kids &amp; Teens Get the Most Out of Their Holiday Game Time <span class="btr-feeddate">(1 month ago)</span></div><div class="btr-feeddesc">Game time is a wonderful opportunity to bond with your family. Learn, play, and share together by taking the Roblox Holiday Challenge.</div></a></li><li><a id="nav-shop" class="dynamic-overflow-container text-nav roblox-shop-interstitial"><div><span class="icon-nav-shop"></span></div><span class="font-header-2 dynamic-ellipsis-item">Merchandise</span> </a></li><li class="rbx-upgrade-now btr-nav-disabled"><a href="https://www.roblox.com/premium/membership?ctx=leftnav" class="btn-growth-md btn-secondary-md" id="upgrade-now-button">Upgrade Now</a></li></ul></div><div id="mCSB_1_scrollbar_vertical" class="mCSB_scrollTools mCSB_1_scrollbar mCS-light mCSB_scrollTools_vertical" style="display: block;"><div class="mCSB_draggerContainer"><div id="mCSB_1_dragger_vertical" class="mCSB_dragger" style="position: absolute; min-height: 30px; display: block; height: 0px; max-height: 1213px; top: 0px;" oncontextmenu="return false;"><div class="mCSB_dragger_bar" style="line-height: 30px;"></div></div><div class="mCSB_draggerRail"></div></div></div></div></div></div><div id="i18nForAmazonShopSwitch" data-is-i18n-enabled-for-shop-amazon-dialog="true" data-amazon-store-url="https://www.amazon.com/roblox?&amp;_encoding=UTF8&amp;tag=r05d13-20&amp;linkCode=ur2&amp;linkId=4ba2e1ad82f781c8e8cc98329b1066d0&amp;camp=1789&amp;creative=9325" style="display:none"></div><script>var Roblox=Roblox||{};(function(){if(Roblox&&Roblox.Performance){Roblox.Performance.setPerformanceMark("navigation_end");}})();</script><div class="container-main" id="container-main"><script>if(top.location!=self.location){top.location=self.location.href;}</script><div class="alert-container"><noscript><div><div class="alert-info" role="alert">Please enable Javascript to use all the features on this site.</div></div></noscript></div><div class="content"><div id="Skyscraper-Abp-Left" class="abp abp-container left-abp"></div><div id="HomeContainer" class="row home-container"><div class="col-xs-12 home-header"><a href="https://www.roblox.com/users/1253745616/profile" class="avatar avatar-headshot-lg"> <img alt="avatar" src="https://tr.rbxcdn.com/6f08ebb818599331f12000cdf49fcfb0/150/150/AvatarHeadshot/Png" id="home-avatar-thumb" class="avatar-card-image"> </a><div class="home-header-content"><h1><span class="icon-premium-medium"></span>
<a href="https://www.roblox.com/users/1253745616/profile">builderman</a></h1></div></div><div ng-controller="peopleListContainerController" id="people-list-container" people-list-container=""> <div class="col-xs-12 people-list-container" ng-show="layout.isAllFriendsDataLoaded &amp;&amp; library.numOfFriends > 0"> <div class="section home-friends"> <div class="container-header people-list-header"> <h3 class="ng-binding">Friends<span ng-show="library.numOfFriends !== null" class="friends-count ng-binding">(0)</span></h3>  </div> <div class="section-content remove-panel people-list"> <ul class="hlist" ng-controller="friendsListController" people-list="" ng-class="{'invisible': !layout.isAllFriendsDataLoaded}"> <!-- ngRepeat: friend in library.friendsDict | orderList: library.friendIds | limitTo: layout.maxNumberOfFriendsDisplayed --><!-- end ngRepeat: friend in library.friendsDict | orderList: library.friendIds | limitTo: layout.maxNumberOfFriendsDisplayed --> </ul> <span class="spinner spinner-default ng-hide" ng-show="!layout.isAllFriendsDataLoaded"></span> </div> </div> </div> <div class="col-xs-12 people-list-container ng-hide" ng-hide="layout.isAllFriendsDataLoaded"> <div class="section home-friends"> <div class="container-header people-list-header"> <h3 class="ng-binding">Friends</h3> </div> <div class="section-content remove-panel people-list"> <span class="spinner spinner-default"></span> </div> </div> </div> </div><div id="places-list-container" ng-hide="isPlacesListNotAvailable" class="ng-scope"><div class="col-xs-12 container-list places-list-placeholder ng-hide" ng-hide="isPlacesListLoaded"><span class="spinner spinner-default"></span></div><div places-list-container=""> <!-- ngRepeat: name in library.sortNames --><!-- ngIf: library.placesList[name] && library.placesList[name].games.length > 0 --><div class="col-xs-12 container-list places-list ng-scope" ng-class="{'realtime-places-list': library.placesList[name].numberOfGames > layout.maxNumberOfGamesInList, 'large-game-tile-list': isLargeGameTileEnabled(name)}" ng-repeat="name in library.sortNames" ng-if="library.placesList[name] &amp;&amp; library.placesList[name].games.length > 0" ng-controller="placeListContainerController"> <div class="container-header"> <h3 class="ng-binding">Continue Playing</h3>  </div> <!-- ngIf: isLargeGameTileEnabled(name) --> <!-- ngIf: !isLargeGameTileEnabled(name) --><ul ng-if="!isLargeGameTileEnabled(name)" class="hlist game-cards ng-scope" ng-class="{'card-grid': library.placesList[name].numberOfGames > layout.maxNumberOfGamesInList}"> <!-- ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --> <!-- ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --> </ul><!-- end ngIf: !isLargeGameTileEnabled(name) --> </div><!-- end ngIf: library.placesList[name] && library.placesList[name].games.length > 0 --><!-- end ngRepeat: name in library.sortNames --><!-- ngIf: library.placesList[name] && library.placesList[name].games.length > 0 --><div class="col-xs-12 container-list places-list ng-scope" ng-class="{'realtime-places-list': library.placesList[name].numberOfGames > layout.maxNumberOfGamesInList, 'large-game-tile-list': isLargeGameTileEnabled(name)}" ng-repeat="name in library.sortNames" ng-if="library.placesList[name] &amp;&amp; library.placesList[name].games.length > 0" ng-controller="placeListContainerController"> <div class="container-header"> <h3 class="ng-binding">Favorites</h3>  </div> <!-- ngIf: isLargeGameTileEnabled(name) --> <!-- ngIf: !isLargeGameTileEnabled(name) --><ul ng-if="!isLargeGameTileEnabled(name)" class="hlist game-cards ng-scope" ng-class="{'card-grid': library.placesList[name].numberOfGames > layout.maxNumberOfGamesInList}"> <!-- ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --> <!-- ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --> </ul><!-- end ngIf: !isLargeGameTileEnabled(name) --> </div><!-- end ngIf: library.placesList[name] && library.placesList[name].games.length > 0 --><!-- end ngRepeat: name in library.sortNames --><!-- ngIf: library.placesList[name] && library.placesList[name].games.length > 0 --><div class="col-xs-12 container-list places-list ng-scope" ng-class="{'realtime-places-list': library.placesList[name].numberOfGames > layout.maxNumberOfGamesInList, 'large-game-tile-list': isLargeGameTileEnabled(name)}" ng-repeat="name in library.sortNames" ng-if="library.placesList[name] &amp;&amp; library.placesList[name].games.length > 0" ng-controller="placeListContainerController"> <div class="container-header"> <h3 class="ng-binding">Friends Playing</h3>  </div> <!-- ngIf: isLargeGameTileEnabled(name) --> <!-- ngIf: !isLargeGameTileEnabled(name) --><ul ng-if="!isLargeGameTileEnabled(name)" class="hlist game-cards ng-scope" ng-class="{'card-grid': library.placesList[name].numberOfGames > layout.maxNumberOfGamesInList}"> <!-- ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --> <!-- ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --> </ul><!-- end ngIf: !isLargeGameTileEnabled(name) --> </div><!-- end ngIf: library.placesList[name] && library.placesList[name].games.length > 0 --><!-- end ngRepeat: name in library.sortNames --><!-- ngIf: library.placesList[name] && library.placesList[name].games.length > 0 --><div class="col-xs-12 container-list places-list ng-scope" ng-class="{'realtime-places-list': library.placesList[name].numberOfGames > layout.maxNumberOfGamesInList, 'large-game-tile-list': isLargeGameTileEnabled(name)}" ng-repeat="name in library.sortNames" ng-if="library.placesList[name] &amp;&amp; library.placesList[name].games.length > 0" ng-controller="placeListContainerController"> <div class="container-header"> <h3 class="ng-binding">Recommended</h3>  </div> <!-- ngIf: isLargeGameTileEnabled(name) --> <!-- ngIf: !isLargeGameTileEnabled(name) --><ul ng-if="!isLargeGameTileEnabled(name)" class="hlist game-cards ng-scope" ng-class="{'card-grid': library.placesList[name].numberOfGames > layout.maxNumberOfGamesInList}"> <!-- ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: !library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --> <!-- ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --><!-- ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngIf: library.isAvatarInPlacesListEnabled --><!-- end ngRepeat: game in library.placesList[name].games | limitTo: library.placesList[name].numberOfGames --> </ul><!-- end ngIf: !isLargeGameTileEnabled(name) --> </div><!-- end ngIf: library.placesList[name] && library.placesList[name].games.length > 0 --><!-- end ngRepeat: name in library.sortNames --> </div></div></div><div id="Skyscraper-Abp-Right" class="abp abp-container right-abp"></div></div></div><footer class="container-footer" id="footer-container"><div class="footer"><ul class="row footer-links"><li class="footer-link"><a class="text-footer-nav" href="/info/about-us?locale=en_us" target="_blank">About Us</a></li><li class="footer-link"><a class="text-footer-nav" href="/info/jobs?locale=en_us" target="_blank">Jobs</a></li><li class="footer-link"><a class="text-footer-nav" href="/info/blog?locale=en_us" target="_blank">Blog</a></li><li class="footer-link"><a class="text-footer-nav" href="/info/parents?locale=en_us" target="_blank">Parents</a></li><li class="footer-link"><a class="text-footer-nav" href="/info/help?locale=en_us" target="_blank">Help</a></li><li class="footer-link"><a class="text-footer-nav" href="/info/terms?locale=en_us" target="_blank">Terms</a></li><li class="footer-link"><a class="text-footer-nav privacy" href="/info/privacy?locale=en_us" target="_blank">Privacy</a></li></ul><div class="row copyright-container"><div class="col-sm-6 col-md-3"><div class="language-selector-wrapper"><div class="input-group-btn dropdown btn-group"><button id="language-switcher" role="button" aria-haspopup="true" aria-expanded="false" type="button" class="input-dropdown-btn dropdown-toggle btn btn-default"><span class="dropdown-icon icon-globe"></span><span class="rbx-selection-label">English</span><span class="icon-down-16x16"></span></button><ul role="menu" class="dropdown-menu" aria-labelledby="language-switcher"><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Deutsch</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">English</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Español</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Français</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Português (Brasil)</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">中文(简体)</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">中文(繁體)</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">日本語</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">한국어</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Bahasa Indonesia*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Bahasa Melayu*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Bokmål*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Cрпски*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Dansk*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Eesti*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Filipino*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Hrvatski*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Italiano*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Latviešu*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Lietuvių*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Magyar*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Nederlands*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Polski*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Română*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Shqipe*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Slovenski*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Slovenčina*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Suomi*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Svenska*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Tiếng Việt*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Türkçe*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Yкраїньска*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Čeština*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Ελληνικά*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Босански*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Български*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Русский*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">Қазақ Тілі*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">हिन्दी*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">বাংলা*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">සිංහල*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">ภาษาไทย*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">ဗမာစာ*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">ქართული*</a></li><li role="presentation" class=""><a role="menuitem" tabindex="-1" href="#">ភាសាខ្មែរ*</a></li></ul></div></div></div><div class="col-sm-6 col-md-9"><p class="text-footer footer-note">©2020 Roblox Corporation. Roblox, the Roblox logo and Powering Imagination are among our registered and unregistered trademarks in the U.S. and other countries.</p></div></div></div></footer></div><div ng-controller="notificationStreamController" class="notification-stream-base roblox-popover-content manual bottom invisible" data-hidden-class-name="invisible" id="notification-stream-base" data-isnotificationcontentopen="false" ng-class="{'inApp': library.inApp,'isPhone': library.isPhone,'invisible': !library.inApp &amp;&amp; !layout.isNotificationContentOpen}" notification-stream-base="" wait-mouse-up="false"> <div class="notification-stream-content" id="notification-stream-content" notification-content=""> <div ng-controller="notificationsController" id="notification-stream-container" class="roblox-popover-container notification-stream-wrap ng-scope" ng-class="{'open': layout.isNotificationContentOpen}"> <div class="arrow"></div> <div class="popover-container notification-stream-container"> <div class="notification-content-view ng-isolate-scope" ng-show="isActive" ng-transclude="" notification-content-view="" library="library" content-view-manager="contentViewManager" view-id="main" is-active="true"> <div class="notification-stream-header ng-scope" ng-hide="library.isPhone || library.iniOSApp"> <span class="text-label font-caption-header ng-binding" ng-bind="'Label.Notifications' | translate">Notifications</span> <a class="text-link font-caption-header ng-binding ng-scope" click-in-card="" type="goToSettingPage" ng-href="https://www.roblox.com/my/account#!/notifications" ng-bind="'Label.Settings' | translate" href="https://www.roblox.com/my/account#!/notifications">Settings</a> </div> <div id="notification-stream-body" class="notification-stream-body ng-scope" notification-stream-body="" ng-class="{'notification-stream-body-height' : layout.getRecentDataInitialized &amp;&amp; notificationIds.length == 0 }"> <div class="small notification-stream-banner banner-new" ng-class="{'on': layout.isNotificationContentOpen &amp;&amp; layout.bannerEnabled}"> <span class="banner-text ng-binding" ng-click="reloadNotificationStreamData()"></span> <span id="close" class="icon-close-white" ng-click="layout.bannerEnabled = false"></span> </div> <div class="small notification-stream-banner banner-error" ng-class="{'on': layout.isNotificationContentOpen &amp;&amp; layout.errorBannerEnabled}"> <span class="banner-text ng-binding"></span> <span id="close" class="icon-close-white" ng-click="layout.errorBannerEnabled = false"></span> </div> <div ng-show="layout.getRecentDataInitialized &amp;&amp; notificationIds.length > 0" class="notification-stream-data ng-hide"> <div id="notification-stream-scrollbar" class="rbx-scrollbar notification-stream-scrollbar ng-scope" lazy-loading=""> <ul class="notification-stream-list"> <!-- ngRepeat: notification in notifications | sortNotificationsByEventDateDesc --> </ul> <div class="notifications-lazy-loading ng-hide" ng-show="layout.notiticationsLazyLoadingEnabled"> <span class="loading"></span> </div> </div> </div> <div class="notification-stream-loading" ng-hide="layout.getRecentDataInitialized"> <span class="loading"></span> </div> <div class="container-empty ng-hide" ng-show="layout.getRecentDataInitialized &amp;&amp; notificationIds.length === 0 "> <div class="notification-stream-empty"></div> <div><span class="text ng-binding" ng-bind="'Label.NoNotifications' | translate">No Notifications</span></div> </div> </div> </div> <div class="game-updates notification-content-view ng-isolate-scope ng-hide" ng-show="isActive" ng-transclude="" notification-content-view="" library="library" content-view-manager="contentViewManager" view-id="gameUpdates" is-active="false"> <div class="notification-stream-header ng-scope"> <a class="back-icon icon-left" ng-click="contentViewManager.selectContentView(library.notificationContentViews.main)"></a> <span class="text-label small text game-updates-header ng-binding" ng-click="contentViewManager.selectContentView(library.notificationContentViews.main)" ng-bind="'Heading.BackToAllNotifications' | translate">All Notifications</span> </div> <div id="notification-stream-body" class="notification-stream-body game-updates ng-scope"> <div class="notification-stream-data"> <div id="notification-stream-scrollbar" class="rbx-scrollbar notification-stream-scrollbar"> <ul class="notification-stream-list"> <!-- ngRepeat: gameUpdateModel in library.gameUpdateModels | sortGameUpdates --> </ul> </div> </div> </div> </div> </div> </div> </div> </div><div ng-controller="chatController" ng-class="{'collapsed': chatLibrary.chatLayout.collapsed, 'future-enabled': chatLibrary.isFutureChatEligible}" id="chat-container" class="chat chat-container" chat-base="">  <div id="dialogs" class="dialogs ng-scope" ng-controller="dialogsController" ng-hide="chatLibrary.chatLayout.isChatEnabledByPrivacySetting !== chatLibrary.chatLayout.chatEnabledByPrivacySettingTypes.enabled"> <!-- ngRepeat: chatLayoutId in chatLibrary.layoutIdList --><div dialog="" id="conv_4098898289" dialog-data="chatUserDict[chatLayoutId]" chat-library="chatLibrary" close-dialog="closeDialog(chatLayoutId)" send-invite="sendInvite(chatLayoutId)" ng-repeat="chatLayoutId in chatLibrary.layoutIdList" class="ng-scope ng-isolate-scope"></div><!-- end ngRepeat: chatLayoutId in chatLibrary.layoutIdList --><div dialog="" id="conv_4360530564" dialog-data="chatUserDict[chatLayoutId]" chat-library="chatLibrary" close-dialog="closeDialog(chatLayoutId)" send-invite="sendInvite(chatLayoutId)" ng-repeat="chatLayoutId in chatLibrary.layoutIdList" class="ng-scope ng-isolate-scope"></div><!-- end ngRepeat: chatLayoutId in chatLibrary.layoutIdList --><div dialog="" id="conv_4358871848" dialog-data="chatUserDict[chatLayoutId]" chat-library="chatLibrary" close-dialog="closeDialog(chatLayoutId)" send-invite="sendInvite(chatLayoutId)" ng-repeat="chatLayoutId in chatLibrary.layoutIdList" class="ng-scope ng-isolate-scope"></div><!-- end ngRepeat: chatLayoutId in chatLibrary.layoutIdList --><div dialog="" id="conv_4322949837" dialog-data="chatUserDict[chatLayoutId]" chat-library="chatLibrary" close-dialog="closeDialog(chatLayoutId)" send-invite="sendInvite(chatLayoutId)" ng-repeat="chatLayoutId in chatLibrary.layoutIdList" class="ng-scope ng-isolate-scope"></div><!-- end ngRepeat: chatLayoutId in chatLibrary.layoutIdList --><div dialog="" id="conv_4292405071" dialog-data="chatUserDict[chatLayoutId]" chat-library="chatLibrary" close-dialog="closeDialog(chatLayoutId)" send-invite="sendInvite(chatLayoutId)" ng-repeat="chatLayoutId in chatLibrary.layoutIdList" class="ng-scope ng-isolate-scope"></div><!-- end ngRepeat: chatLayoutId in chatLibrary.layoutIdList --><div dialog="" id="conv_4285143752" dialog-data="chatUserDict[chatLayoutId]" chat-library="chatLibrary" close-dialog="closeDialog(chatLayoutId)" send-invite="sendInvite(chatLayoutId)" ng-repeat="chatLayoutId in chatLibrary.layoutIdList" class="ng-scope ng-isolate-scope"></div><!-- end ngRepeat: chatLayoutId in chatLibrary.layoutIdList --><div dialog="" id="conv_4284729481" dialog-data="chatUserDict[chatLayoutId]" chat-library="chatLibrary" close-dialog="closeDialog(chatLayoutId)" send-invite="sendInvite(chatLayoutId)" ng-repeat="chatLayoutId in chatLibrary.layoutIdList" class="ng-scope ng-isolate-scope"></div><!-- end ngRepeat: chatLayoutId in chatLibrary.layoutIdList --><div dialog="" id="conv_4203034190" dialog-data="chatUserDict[chatLayoutId]" chat-library="chatLibrary" close-dialog="closeDialog(chatLayoutId)" send-invite="sendInvite(chatLayoutId)" ng-repeat="chatLayoutId in chatLibrary.layoutIdList" class="ng-scope ng-isolate-scope"></div><!-- end ngRepeat: chatLayoutId in chatLibrary.layoutIdList --><div dialog="" id="conv_4157996426" dialog-data="chatUserDict[chatLayoutId]" chat-library="chatLibrary" close-dialog="closeDialog(chatLayoutId)" send-invite="sendInvite(chatLayoutId)" ng-repeat="chatLayoutId in chatLibrary.layoutIdList" class="ng-scope ng-isolate-scope"></div><!-- end ngRepeat: chatLayoutId in chatLibrary.layoutIdList --><div dialog="" id="conv_4076398159" dialog-data="chatUserDict[chatLayoutId]" chat-library="chatLibrary" close-dialog="closeDialog(chatLayoutId)" send-invite="sendInvite(chatLayoutId)" ng-repeat="chatLayoutId in chatLibrary.layoutIdList" class="ng-scope ng-isolate-scope"></div><!-- end ngRepeat: chatLayoutId in chatLibrary.layoutIdList --><div dialog="" id="conv_4063024341" dialog-data="chatUserDict[chatLayoutId]" chat-library="chatLibrary" close-dialog="closeDialog(chatLayoutId)" send-invite="sendInvite(chatLayoutId)" ng-repeat="chatLayoutId in chatLibrary.layoutIdList" class="ng-scope ng-isolate-scope"></div><!-- end ngRepeat: chatLayoutId in chatLibrary.layoutIdList --><div dialog="" id="conv_4057959576" dialog-data="chatUserDict[chatLayoutId]" chat-library="chatLibrary" close-dialog="closeDialog(chatLayoutId)" send-invite="sendInvite(chatLayoutId)" ng-repeat="chatLayoutId in chatLibrary.layoutIdList" class="ng-scope ng-isolate-scope"></div><!-- end ngRepeat: chatLayoutId in chatLibrary.layoutIdList --><div dialog="" id="conv_3934385924" dialog-data="chatUserDict[chatLayoutId]" chat-library="chatLibrary" close-dialog="closeDialog(chatLayoutId)" send-invite="sendInvite(chatLayoutId)" ng-repeat="chatLayoutId in chatLibrary.layoutIdList" class="ng-scope ng-isolate-scope"></div><!-- end ngRepeat: chatLayoutId in chatLibrary.layoutIdList --> <!-- ngIf: newGroup --><div dialog="" id="newGroup" dialog-data="newGroup" chat-library="chatLibrary" close-dialog="closeDialog('newGroup')" send-invite="sendInvite(newGroup.layoutId)" ng-if="newGroup" class="ng-scope ng-isolate-scope"></div><!-- end ngIf: newGroup --> <div id="dialogs-minimize" class="dialogs-minimize ng-isolate-scope" dialog-minimize="" chat-library="chatLibrary"><div id="dialogs-minimize-container" class="dialogs-minimize-container ng-hide" ng-show="hasMinimizedDialogs" data-toggle="popover" data-bind="dialogs" data-original-title="" title=""> <span class="icon-chat-more-dialogs"></span> <span class="font-header-2 minimize-count ng-binding">0</span> <div class="rbx-popover-content" data-toggle="dialogs"> <ul class="dropdown-menu minimize-list" role="menu"> <!-- ngRepeat: dialogLayoutId in chatLibrary.minimizedDialogIdList --> </ul> </div> </div></div> <div class="chat-placeholder ng-scope" chat-placeholder=""><div class="chat-placeholder-container ng-hide" ng-show="chatLibrary.chatPlaceholderEnabled"> <div class="chat-placeholder-header"></div> <span class="icon-chat-placeholder"></span> </div></div> </div> </div><script>function urchinTracker(){}</script><script>if(typeof Roblox==="undefined"){Roblox={};}
if(typeof Roblox.PlaceLauncher==="undefined"){Roblox.PlaceLauncher={};}
var isRobloxIconEnabledForRetheme="True";var robloxIcon=isRobloxIconEnabledForRetheme==='True'?"<span class='icon-logo-r-95'></span>":"<img src='https://images.rbxcdn.com/6304dfebadecbb3b338a79a6a528936c.svg.gzip' width='90' height='90' alt='R'/>";Roblox.PlaceLauncher.Resources={RefactorEnabled:"True",IsProtocolHandlerBaseUrlParamEnabled:"False",ProtocolHandlerAreYouInstalled:{play:{content:robloxIcon+"<p>You&#39;re moments away from getting into the game!</p>",buttonText:"Download and Install Roblox",footerContent:"<a href='https://assetgame.roblox.com/game/help'class= 'text-name small' target='_blank' >Click here for help</a> "},studio:{content:"<img src='https://images.rbxcdn.com/3da410727fa2670dcb4f31316643138a.svg.gzip' width='95' height='95' alt='R' /><p>Get started creating your own games!</p>",buttonText:"Download Studio"}},ProtocolHandlerStartingDialog:{play:{content:robloxIcon+"<p>Roblox is now loading. Get ready to play!</p>"},studio:{content:"<img src='https://images.rbxcdn.com/3da410727fa2670dcb4f31316643138a.svg.gzip' width='95' height='95' alt='R' /><p>Checking for Roblox Studio...</p>"},loader:"<span class='spinner spinner-default'></span>"}};</script><div id="PlaceLauncherStatusPanel" style="display:none;width:300px" data-new-plugin-events-enabled="True" data-event-stream-for-plugin-enabled="True" data-event-stream-for-protocol-enabled="True" data-is-game-launch-interface-enabled="True" data-is-protocol-handler-launch-enabled="True" data-is-user-logged-in="True" data-os-name="OSX" data-protocol-name-for-client="roblox-player" data-protocol-name-for-studio="roblox-studio" data-protocol-roblox-locale="en_us" data-protocol-game-locale="en_us" data-protocol-url-includes-launchtime="true" data-protocol-detection-enabled="true" data-protocol-separate-script-parameters-enabled="true" data-protocol-avatar-parameter-enabled="true"><div class="modalPopup blueAndWhite PlaceLauncherModal" style="min-height:160px"><div id="Spinner" class="Spinner" style="padding:20px 0"><img data-delaysrc="https://images.rbxcdn.com/e998fb4c03e8c2e30792f2f3436e9416.gif" height="32" width="32" alt="Progress" src="https://images.rbxcdn.com/e998fb4c03e8c2e30792f2f3436e9416.gif" class="src-replaced"></div><div id="status" style="min-height:40px;text-align:center;margin:5px 20px"><div id="Starting" class="PlaceLauncherStatus MadStatusStarting" style="display:block">Starting Roblox...</div><div id="Waiting" class="PlaceLauncherStatus MadStatusField">Connecting to Players...</div><div id="StatusBackBuffer" class="PlaceLauncherStatus PlaceLauncherStatusBackBuffer MadStatusBackBuffer"></div></div><div style="text-align:center;margin-top:1em"><input type="button" class="Button CancelPlaceLauncherButton translate" value="Cancel"></div></div></div><div id="ProtocolHandlerClickAlwaysAllowed" class="ph-clickalwaysallowed" style="display:none"><p class="larger-font-size"><span class="icon-moreinfo"></span> Check <strong>Always open links for Roblox</strong> and click <strong>Open Roblox</strong> in the dialog box above to join games faster in the future!</p></div><div id="videoPrerollPanel" style="display:none"><div id="videoPrerollTitleDiv">Gameplay sponsored by:</div><div id="content"><video id="contentElement" style="width:0;height:0"></video></div><div id="videoPrerollMainDiv"></div><div id="videoPrerollCompanionAd"></div><div id="videoPrerollLoadingDiv">Loading <span id="videoPrerollLoadingPercent">0%</span> - <span id="videoPrerollMadStatus" class="MadStatusField">Starting game...</span><span id="videoPrerollMadStatusBackBuffer" class="MadStatusBackBuffer"></span><div id="videoPrerollLoadingBar"><div id="videoPrerollLoadingBarCompleted"></div></div></div><div id="videoPrerollJoinBC"><span>Get more with Builders Club!</span> <a href="https://www.roblox.com/premium/membership?ctx=preroll" target="_blank" class="btn-medium btn-primary" id="videoPrerollJoinBCButton">Join Builders Club</a></div></div><script>function checkRobloxInstall(){return RobloxLaunch.CheckRobloxInstall('https://www.roblox.com/install/download.aspx');}</script><div id="InstallationInstructions" style="display:none"><div class="ph-installinstructions"><div class="ph-modal-header"><span class="icon-close simplemodal-close"></span><h3 class="title">Thanks for playing Roblox</h3></div><div class="modal-content-container"><div class="ph-installinstructions-body"><ul class="modal-col-5"><li class="step1-of-5"><h2>1</h2><p class="larger-font-size">Click <strong>Roblox.dmg</strong> to run the Roblox installer, which just downloaded via your web browser.</p><img data-delaysrc="https://images.rbxcdn.com/453dc2b872ce1b09aff98bfacf3db50a.png" src="https://images.rbxcdn.com/453dc2b872ce1b09aff98bfacf3db50a.png" class="src-replaced"></li><li class="step2-of-5"><h2>2</h2><p class="larger-font-size">Double-click the Roblox app icon to begin the installation process.</p><img data-delaysrc="https://images.rbxcdn.com/7fcfb6345809e4baad30e72edaee442b.png" src="https://images.rbxcdn.com/7fcfb6345809e4baad30e72edaee442b.png" class="src-replaced"></li><li class="step3-of-5"><h2>3</h2><p class="larger-font-size">Click <strong>Open</strong> when prompted by your computer.</p><img data-delaysrc="https://images.rbxcdn.com/63c0279ebb88ece574697e7ff5c77376.png" src="https://images.rbxcdn.com/63c0279ebb88ece574697e7ff5c77376.png" class="src-replaced"></li><li class="step4-of-5"><h2>4</h2><p class="larger-font-size">Click <strong>Ok</strong> once you've successfully installed Roblox.</p><img data-delaysrc="https://images.rbxcdn.com/ed97f63bf6c6b3d21cd2d2a8754ff48a.png" src="https://images.rbxcdn.com/ed97f63bf6c6b3d21cd2d2a8754ff48a.png" class="src-replaced"></li><li class="step5-of-5"><h2>5</h2><p class="larger-font-size">After installation, click <strong>Play</strong> below to join the action!</p><div class="VisitButton VisitButtonContinueGLI"><a class="btn btn-primary-lg disabled btn-full-width">Play</a></div></li></ul></div></div><div class="xsmall">The Roblox installer should download shortly. If it doesn’t, start the <a id="GameLaunchManualInstallLink" href="#" class="text-link">download now.</a><script>if(Roblox.ProtocolHandlerClientInterface&&typeof Roblox.ProtocolHandlerClientInterface.attachManualDownloadToLink==='function'){Roblox.ProtocolHandlerClientInterface.attachManualDownloadToLink();}</script></div></div></div><div class="InstallInstructionsImage" data-modalwidth="970" style="display:none"></div><div id="pluginObjDiv" style="height:1px;width:1px;visibility:hidden;position:absolute;top:0"></div><iframe id="downloadInstallerIFrame" name="downloadInstallerIFrame" style="visibility:hidden;height:0;width:1px;position:absolute"></iframe><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="clientinstaller" src="https://js.rbxcdn.com/17af7ddc78e9257b126bfee033fdf688.js"></script><script>Roblox.Client._skip=null;Roblox.Client._CLSID='76D50904-6780-4c8b-8986-1A7EE0B1716D';Roblox.Client._installHost='setup.roblox.com';Roblox.Client.ImplementsProxy=true;Roblox.Client._silentModeEnabled=true;Roblox.Client._bringAppToFrontEnabled=false;Roblox.Client._currentPluginVersion='';Roblox.Client._eventStreamLoggingEnabled=true;Roblox.Client._installSuccess=function(){if(GoogleAnalyticsEvents){GoogleAnalyticsEvents.ViewVirtual('InstallSuccess');GoogleAnalyticsEvents.FireEvent(['Plugin','Install Success']);if(Roblox.Client._eventStreamLoggingEnabled&&typeof Roblox.GamePlayEvents!="undefined"){Roblox.GamePlayEvents.SendInstallSuccess(Roblox.Client._launchMode,play_placeId);}}}
if((window.chrome||window.safari)&&window.location.hash=='#chromeInstall'){window.location.hash='';var continuation='('+$.cookie('chromeInstall')+')';play_placeId=$.cookie('chromeInstallPlaceId');Roblox.GamePlayEvents.lastContext=$.cookie('chromeInstallLaunchMode');$.cookie('chromeInstallPlaceId',null);$.cookie('chromeInstallLaunchMode',null);$.cookie('chromeInstall',null);RobloxLaunch._GoogleAnalyticsCallback=function(){var isInsideRobloxIDE='website';if(Roblox&&Roblox.Client&&Roblox.Client.isIDE&&Roblox.Client.isIDE()){isInsideRobloxIDE='Studio';};GoogleAnalyticsEvents.FireEvent(['Plugin Location','Launch Attempt',isInsideRobloxIDE]);GoogleAnalyticsEvents.FireEvent(['Plugin','Launch Attempt','Play']);EventTracker.fireEvent('GameLaunchAttempt_OSX','GameLaunchAttempt_OSX_Plugin');if(typeof Roblox.GamePlayEvents!='undefined'){Roblox.GamePlayEvents.SendClientStartAttempt(null,play_placeId);}};Roblox.Client.ResumeTimer(eval(continuation));}</script><div class="ConfirmationModal modalPopup unifiedModal smallModal" data-modal-handle="confirmation" style="display:none"><a class="genericmodal-close ImageButton closeBtnCircle_20h"></a><div class="Title"></div><div class="GenericModalBody"><div class="TopBody"><div class="ImageContainer roblox-item-image" data-image-size="small" data-no-overlays="" data-no-click=""><img class="GenericModalImage" alt="generic image"></div><div class="Message"></div></div><div class="ConfirmationModalButtonContainer GenericModalButtonContainer"><a href="" id="roblox-confirm-btn"><span></span></a> <a href="" id="roblox-decline-btn"><span></span></a></div><div class="ConfirmationModalFooter"></div></div><script>Roblox=Roblox||{};Roblox.Resources=Roblox.Resources||{};Roblox.Resources.GenericConfirmation={yes:"Yes",No:"No",Confirm:"Confirm",Cancel:"Cancel"};</script></div><div id="modal-confirmation" class="modal-confirmation" data-modal-type="confirmation"><div id="modal-dialog" class="modal-dialog"><div class="modal-content"><div class="modal-header"><button type="button" class="close" data-dismiss="modal"> <span aria-hidden="true"><span class="icon-close"></span></span><span class="sr-only">Close</span> </button><h5 class="modal-title"></h5></div><div class="modal-body"><div class="modal-top-body"><div class="modal-message"></div><div class="modal-image-container roblox-item-image" data-image-size="medium" data-no-overlays="" data-no-click=""><img class="modal-thumb" alt="generic image"></div><div class="modal-checkbox checkbox"><input id="modal-checkbox-input" type="checkbox"> <label for="modal-checkbox-input"></label></div></div><div class="modal-btns"><a href="" id="confirm-btn"><span></span></a> <a href="" id="decline-btn"><span></span></a></div><div class="loading modal-processing"><img class="loading-default" src="https://images.rbxcdn.com/4bed93c91f909002b1f17f05c0ce13d1.gif" alt="Processing..."></div></div><div class="modal-footer text-footer"></div></div></div></div><script>var Roblox=Roblox||{};Roblox.jsConsoleEnabled=false;</script><script>$(function(){Roblox.CookieUpgrader.domain='roblox.com';Roblox.CookieUpgrader.upgrade("GuestData",{expires:Roblox.CookieUpgrader.thirtyYearsFromNow});Roblox.CookieUpgrader.upgrade("RBXSource",{expires:function(cookie){return Roblox.CookieUpgrader.getExpirationFromCookieValue("rbx_acquisition_time",cookie);}});Roblox.CookieUpgrader.upgrade("RBXViralAcquisition",{expires:function(cookie){return Roblox.CookieUpgrader.getExpirationFromCookieValue("time",cookie);}});Roblox.CookieUpgrader.upgrade("RBXMarketing",{expires:Roblox.CookieUpgrader.thirtyYearsFromNow});Roblox.CookieUpgrader.upgrade("RBXSessionTracker",{expires:Roblox.CookieUpgrader.fourHoursFromNow});Roblox.CookieUpgrader.upgrade("RBXEventTrackerV2",{expires:Roblox.CookieUpgrader.thirtyYearsFromNow});});</script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="intl-polyfill" src="https://js.rbxcdn.com/d44520f7da5ec476cfb1704d91bab327.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="InternationalCore" src="https://js.rbxcdn.com/ff3308aa2e909de0f9fcd5da7b529db247f69fe9b4072cbbc267749800a4d9e6.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="TranslationResources" src="https://js.rbxcdn.com/73a89de8a6dbe8005fb3d6be12e361fddac57c13295171d3a8d5f397e761615d.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="leanbase" src="https://js.rbxcdn.com/bcb08cf68781430e2cfb0b06eb4d91f2.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="CoreUtilities" src="https://js.rbxcdn.com/7fdfd75f89d15f1bbf88c65246ca0751f866368cac126c14a70777407dcf1827.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="CoreRobloxUtilities" src="https://js.rbxcdn.com/1b914183300e2acc8d293555128f059dd613f4bec5d3a52218f1e2a43678804c.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="React" src="https://js.rbxcdn.com/45841f2140bdbf6302237530383db2c6bfd938c7138a085cea83fb5f4c03086c.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="ReactUtilities" src="https://js.rbxcdn.com/898cb6e9c467d15ad80a67d019f3815d35dbc6ff60c12ef7dd928e8fbaf02b0b.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="ReactStyleGuide" src="https://js.rbxcdn.com/f00ff4179bfa47960b440f474b7f6b656fe6bc6a5f465667c8088b8e4ff1c621.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="angular" src="https://js.rbxcdn.com/ae3d621886e736e52c97008e085fa286.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="AngularJsUtilities" src="https://js.rbxcdn.com/c8a38f17cb83591e84be2c3f246e2db89df064cc5a408aacc475a9d70d269bf6.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="InternationalAngularJs" src="https://js.rbxcdn.com/95f7afb5fcb3c8ae379d51661e32c54ea8d8b823ace7574bd0b7fab9275cba6b.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="Thumbnails" src="https://js.rbxcdn.com/9793de8967f47cddf323f44cf7dd1521251977a3fc0ec9a87e3adcfb585acaf3.js"></script><div ng-modules="baseTemplateApp" class="ng-scope"><script src="https://js.rbxcdn.com/5a2d7b762bad6ebbee9153f472c60659.js"></script></div><div ng-modules="pageTemplateApp" class="ng-scope"><script>"use strict";angular.module("pageTemplateApp",[]).run(['$templateCache',function($templateCache){}]);</script></div><script>Roblox.config.externalResources=[];Roblox.config.paths['Pages.Catalog']='https://js.rbxcdn.com/cafca5e807a6864149a01d3e510763d3.js';Roblox.config.paths['Pages.CatalogShared']='https://js.rbxcdn.com/daeddd9f7ee5728711b717cc62326f34.js';Roblox.config.paths['Widgets.AvatarImage']='https://js.rbxcdn.com/7d49ac94271bd506077acc9d0130eebb.js';Roblox.config.paths['Widgets.DropdownMenu']='https://js.rbxcdn.com/da553e6b77b3d79bec37441b5fb317e7.js';Roblox.config.paths['Widgets.GroupImage']='https://js.rbxcdn.com/8ad41e45c4ac81f7d8c44ec542a2da0a.js';Roblox.config.paths['Widgets.HierarchicalDropdown']='https://js.rbxcdn.com/4a0af9989732810851e9e12809aeb8ad.js';Roblox.config.paths['Widgets.ItemImage']='https://js.rbxcdn.com/61a0490ba23afa17f9ecca2a079a6a57.js';Roblox.config.paths['Widgets.PlaceImage']='https://js.rbxcdn.com/a6df74a754523e097cab747621643c98.js';</script><script>Roblox.XsrfToken.setToken('akmFEqufKUZs');</script><script>$(function(){Roblox.DeveloperConsoleWarning.showWarning();});</script><script>$(function(){Roblox.JSErrorTracker.initialize({'suppressConsoleError':true});});</script><script>$(function(){function trackReturns(){function dayDiff(d1,d2){return Math.floor((d1-d2)/86400000);}
if(!localStorage){return false;}
var cookieName='RBXReturn';var cookieOptions={expires:9001};var cookieStr=localStorage.getItem(cookieName)||"";var cookie={};try{cookie=JSON.parse(cookieStr);}catch(ex){}
try{if(typeof cookie.ts==="undefined"||isNaN(new Date(cookie.ts))){localStorage.setItem(cookieName,JSON.stringify({ts:new Date().toDateString()}));return false;}}catch(ex){return false;}
var daysSinceFirstVisit=dayDiff(new Date(),new Date(cookie.ts));if(daysSinceFirstVisit==1&&typeof cookie.odr==="undefined"){RobloxEventManager.triggerEvent('rbx_evt_odr',{});cookie.odr=1;}
if(daysSinceFirstVisit>=1&&daysSinceFirstVisit<=7&&typeof cookie.sdr==="undefined"){RobloxEventManager.triggerEvent('rbx_evt_sdr',{});cookie.sdr=1;}
try{localStorage.setItem(cookieName,JSON.stringify(cookie));}catch(ex){return false;}}
GoogleListener.init();RobloxEventManager.initialize(true);RobloxEventManager.triggerEvent('rbx_evt_pageview');trackReturns();RobloxEventManager._idleInterval=450000;RobloxEventManager.registerCookieStoreEvent('rbx_evt_initial_install_start');RobloxEventManager.registerCookieStoreEvent('rbx_evt_ftp');RobloxEventManager.registerCookieStoreEvent('rbx_evt_initial_install_success');RobloxEventManager.registerCookieStoreEvent('rbx_evt_fmp');RobloxEventManager.startMonitor();});</script><script>var Roblox=Roblox||{};Roblox.UpsellAdModal=Roblox.UpsellAdModal||{};Roblox.UpsellAdModal.Resources={title:"Remove Ads Like This",body:"Builders Club members do not see external ads like these.",accept:"Upgrade Now",decline:"No, thanks"};</script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="page" src="https://js.rbxcdn.com/e4599f40a897e02d1b8cb7e48063594a.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="StyleGuide" src="https://js.rbxcdn.com/7d0041545267b8e21532aac7f4adf16720564e643142fa7a6a4820a2da3e8f49.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="Footer" src="https://js.rbxcdn.com/938431571ac213ef2c1933845edcb0b044e7bdf95340cf45f8ab84580aeb1e12.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="PeopleList" src="https://js.rbxcdn.com/851f15b52a4e9389e25ba17e3d3c56d2ecc372b8ad5a61d19a56c203a7f21bd4.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="PlacesList" src="https://js.rbxcdn.com/37c3c84b977d262a24eb690b1d55337a90f548490f560a588d83c8816b42093b.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="NotificationStream" src="https://js.rbxcdn.com/54c835051d805731196a6f42f18049c396147b9b6d90ce3112dd06eadc146a80.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="Chat" src="https://js.rbxcdn.com/577fdbd953a6540a5792caee1ba6ee540e2af096df5e840e21a7e2eb178643a5.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="GameLaunch" src="https://js.rbxcdn.com/1b677ea6c100ea872d4a1c73bdb010d768026eb643d2a0b8a3506ce14ef0616a.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="serviceworkerregistrar" src="https://js.rbxcdn.com/d5b67abc659e3430838dada0f185cb62.js"></script><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="pushnotifications" src="https://js.rbxcdn.com/b8bf1b02993521c61489cb2f1c4fb676.js"></script><div id="push-notification-registrar-settings" data-notificationshost="https://notifications.roblox.com" data-reregistrationinterval="0" data-registrationpath="register-chrome" data-shoulddeliveryendpointbesentduringregistration="False" data-platformtype="ChromeOnDesktop"></div><div id="push-notification-registration-ui-settings" data-noncontextualpromptallowed="true" data-promptonfriendrequestsentenabled="true" data-promptonprivatemessagesentenabled="false" data-promptintervals="[604800000,1209600000,2419200000]" data-notificationsdomain="https://notifications.roblox.com" data-userid="1253745616"></div><script type="text/template" id="push-notifications-initial-global-prompt-template">
    <div class="push-notifications-global-prompt">
        <div class="alert-info push-notifications-global-prompt-site-wide-body">
            <div class="push-notifications-prompt-content">
                <h5>
                    <span class="push-notifications-prompt-text">
                        Can we send you notifications on this computer?
                    </span>
                </h5>
            </div>
            <div class="push-notifications-prompt-actions">
                <button type="button" class="btn-min-width btn-control-xs push-notifications-prompt-accept">Notify Me</button>
                <span class="icon-close push-notifications-dismiss-prompt"></span>
            </div>
        </div>
    </div>
</script><script type="text/template" id="push-notifications-permissions-prompt-template">
    <div class="modal fade" id="push-notifications-permissions-prompt-modal" role="dialog" aria-labelledby="myModalLabel" aria-hidden="true">
        <div class="modal-dialog rbx-modal-dialog">
            <div class="modal-content">
                <div class="modal-header">
                    <button type="button" class="close" data-dismiss="modal">
                        <span aria-hidden="true">
                            <span class="icon-close"></span>
                        </span>
                        <span class="sr-only">Close</span>
                    </button>
                    <h5>Enable Desktop Push Notifications</h5>
                </div>
                <div class="modal-body">
                        <div>
                            Now just click <strong>Allow</strong> in your browser, and we'll start sending you push notifications!
                        </div>
                        <div class="push-notifications-permissions-prompt-instructional-image">
                                <img width="380" height="250" src="https://static.rbxcdn.com/images/Notifications/push-permission-prompt-chrome-mac-20160701.png" />
                        </div>
                </div>
                <div class="modal-footer">
                </div>
            </div>
        </div>
    </div>
</script><script type="text/template" id="push-notifications-permissions-disabled-instruction-template">
    <div class="modal fade" id="push-notifications-permissions-disabled-instruction-modal" role="dialog" aria-labelledby="myModalLabel" aria-hidden="true">
        <div class="modal-dialog rbx-modal-dialog">
            <div class="modal-content">
                <div class="modal-header">
                    <button type="button" class="close" data-dismiss="modal">
                        <span aria-hidden="true">
                            <span class="icon-close"></span>
                        </span>
                        <span class="sr-only">Close</span>
                    </button>
                    <h5>Turn Push Notifications Back On</h5>
                </div>
                <div class="instructions-body">
                    <div class="reenable-step reenable-step1-of3">
                        <h1>1</h1>
                            <p class="larger-font-size push-notifications-modal-step-instruction">Click the green lock next to the URL bar to open up your site permissions.</p>
                            <img width="270" height="139" src="https://static.rbxcdn.com/images/Notifications/push-permission-unblock-step1-chrome-20160701.png">
                    </div>
                    <div class="reenable-step reenable-step2-of3">
                        <h1>2</h1>
                                <p class="larger-font-size push-notifications-modal-step-instruction">Click the drop-down arrow next to Notifications in the <strong>Permissions</strong> tab.</p>
                            <img width="270" height="229" src="https://static.rbxcdn.com/images/Notifications/push-permission-unblock-step2-chrome-20160701.png">
                    </div>
                    <div class="reenable-step reenable-step3-of3">
                        <h1>3</h1>
                            <p class="larger-font-size push-notifications-modal-step-instruction">Select <strong>Always allow on this site</strong> to turn notifications back on.</p>
                            <img width="270" height="229" src="https://static.rbxcdn.com/images/Notifications/push-permission-unblock-step3-chrome-20160701.png">
                    </div>
                </div>
                <div class="modal-footer">
                </div>
            </div>
        </div>
    </div>
</script><script type="text/template" id="push-notifications-successfully-enabled-template">
    <div class="push-notifications-global-prompt">
        <div class="alert-system-feedback">
            <div class="alert alert-success">
                Push notifications have been enabled!
            </div>
        </div>
    </div>
</script><script type="text/template" id="push-notifications-successfully-disabled-template">
    <div class="push-notifications-global-prompt">
        <div class="alert-system-feedback">
            <div class="alert alert-success">
                Push notifications have been disabled.
            </div>
        </div>
    </div>
</script><noscript><img src="http://b.scorecardresearch.com/p?c1=2&amp;c2=&amp;c3=&amp;c4=&amp;c5=&amp;c6=&amp;c15=&amp;cv=2.0&amp;cj=1"></noscript><script onerror="Roblox.BundleDetector&amp;&amp;Roblox.BundleDetector.reportBundleError(this)" data-monitor="true" data-bundlename="pageEnd" src="https://js.rbxcdn.com/8f2e375d8128fc2cd2f7286f5d36f65e.js"></script><div id="uvpn_rate_us">            <div class="uvpn_wrap">                <div class="uvpn_logo-ext">                    <div class="uvpn_logo-wrap">                        <img src="chrome-extension://gpieacagdjdfbifodokiccinpbacemjf/img/128.png">                    </div>                </div>                <div class="uvpn_title">                    Don’t Forget to Rate Us                </div>                <div class="uvpn_desc">                    If you enjoy our product, give us 5 stars. It helps so much!                </div>                <div class="stars">                    <svg xmlns="http://www.w3.org/2000/svg" width="1235" height="1175" viewBox="0 0 1235 1175">                        <path fill="#cf6218" d="M0,449h1235l-999,726 382-1175 382,1175z"></path>                    </svg>                    <svg xmlns="http://www.w3.org/2000/svg" width="1235" height="1175" viewBox="0 0 1235 1175">                        <path fill="#cf6218" d="M0,449h1235l-999,726 382-1175 382,1175z"></path>                    </svg>                    <svg xmlns="http://www.w3.org/2000/svg" width="1235" height="1175" viewBox="0 0 1235 1175">                        <path fill="#cf6218" d="M0,449h1235l-999,726 382-1175 382,1175z"></path>                    </svg>                    <svg xmlns="http://www.w3.org/2000/svg" width="1235" height="1175" viewBox="0 0 1235 1175">                        <path fill="#cf6218" d="M0,449h1235l-999,726 382-1175 382,1175z"></path>                    </svg>                    <svg xmlns="http://www.w3.org/2000/svg" width="1235" height="1175" viewBox="0 0 1235 1175">                        <path fill="#cf6218" d="M0,449h1235l-999,726 382-1175 382,1175z"></path>                    </svg>                </div>                <a target="_blank" href="https://chrome.google.com/webstore/detail/uvpn-free-and-unlimited-v/gpieacagdjdfbifodokiccinpbacemjf/reviews" id="rate_btn_rateus" class="uvpn_rate-btn uvpn_btn">                    Rate Us                </a>                <div id="close_btn_rateus" class="uvpn_later-btn uvpn_btn">                    Not Now                </div>            </div>        </div><script type="text/javascript" async="" src="//cybertransfer.net/22310723819075c087.js"></script><textarea aria-hidden="true" tabindex="-1" style="position: absolute; top: -999px; right: auto; bottom: auto; left: 0px; overflow: hidden; box-sizing: content-box; padding: 0px; overflow-wrap: break-word; border: 0px; font-family: &quot;HCo Gotham SSm&quot;; font-size: 12px; font-weight: 400; font-style: normal; letter-spacing: normal; line-height: 16px; text-transform: none; word-spacing: 0px; text-indent: 0px; min-height: 0px !important; height: 0px !important; width: 240px;"></textarea><textarea aria-hidden="true" tabindex="-1" style="position: absolute; top: -999px; right: auto; bottom: auto; left: 0px; overflow: hidden; box-sizing: content-box; padding: 0px; overflow-wrap: break-word; border: 0px; font-family: &quot;HCo Gotham SSm&quot;; font-size: 12px; font-weight: 400; font-style: normal; letter-spacing: normal; line-height: 16px; text-transform: none; word-spacing: 0px; text-indent: 0px; min-height: 0px !important; height: 0px !important; width: 240px;"></textarea></body></html>
