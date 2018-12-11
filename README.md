# wxjava

package com.esb.common;

import java.io.BufferedReader;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.OutputStream;
import java.net.ConnectException;
import java.net.URL;
import java.security.cert.CertificateException;
import java.security.cert.X509Certificate;
import java.text.SimpleDateFormat;
import java.util.Date;

import javax.net.ssl.HttpsURLConnection;
import javax.net.ssl.SSLContext;
import javax.net.ssl.SSLSocketFactory;
import javax.net.ssl.TrustManager;
import javax.net.ssl.X509TrustManager;

import org.apache.log4j.Logger;
import org.springframework.util.StringUtils;



public class WxUtil {
	private static Logger log = Logger.getLogger(WxUtil.class);

	static String SEND_MSG_TEMPLATE_URL="https://api.weixin.qq.com/cgi-bin/message/template/send?access_token=ACCESS_TOKEN";
	static String TOKEN_URL="https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid=APPID&secret=APPSECRET";
	/**
	 * 发送https请求
	 * @param requestUrl 请求地址
	 * @param requestMethod 请求方式（GET、POST）
	 * @param outputStr 提交的数据
	 * @return 返回微信服务器响应的信息
	 */
	public static String httpsRequest(String requestUrl, String requestMethod, String outputStr) {
		try {
			// 创建SSLContext对象，并使用我们指定的信任管理器初始化
			TrustManager[] tm = {
					new X509TrustManager(){
						// 检查客户端证书
						public void checkClientTrusted(X509Certificate[] chain, String authType) throws CertificateException {
						}
						// 检查服务器端证书
						public void checkServerTrusted(X509Certificate[] chain, String authType) throws CertificateException {
						}
						// 返回受信任的X509证书数组
						public X509Certificate[] getAcceptedIssuers() {
							return null;
						}
				}
			};
			SSLContext sslContext = SSLContext.getInstance("SSL", "SunJSSE");
			sslContext.init(null, tm, new java.security.SecureRandom());
			// 从上述SSLContext对象中得到SSLSocketFactory对象
			SSLSocketFactory ssf = sslContext.getSocketFactory();
			URL url = new URL(requestUrl);
			HttpsURLConnection conn = (HttpsURLConnection) url.openConnection();
			conn.setSSLSocketFactory(ssf);
			conn.setDoOutput(true);
			conn.setDoInput(true);
			conn.setUseCaches(false);
			// 设置请求方式（GET/POST）
			conn.setRequestMethod(requestMethod);
			conn.setRequestProperty("content-type", "application/x-www-form-urlencoded"); 
			// 当outputStr不为null时向输出流写数据
			if (null != outputStr) {
				OutputStream outputStream = conn.getOutputStream();
				// 注意编码格式
				outputStream.write(outputStr.getBytes("UTF-8"));
				outputStream.close();
			}
			// 从输入流读取返回内容
			InputStream inputStream = conn.getInputStream();
			InputStreamReader inputStreamReader = new InputStreamReader(inputStream, "utf-8");
			BufferedReader bufferedReader = new BufferedReader(inputStreamReader);
			String str = null;
			StringBuffer buffer = new StringBuffer();
			while ((str = bufferedReader.readLine()) != null) {
				buffer.append(str);
			}
			// 释放资源
			bufferedReader.close();
			inputStreamReader.close();
			inputStream.close();
			inputStream = null;
			conn.disconnect();
			return buffer.toString();
		} catch (ConnectException ce) {
			log.error("连接超时：{}", ce);
		} catch (Exception e) {
			e.printStackTrace();
			log.error("https请求异常：{}", e);
		}
		return null;
	}
	/**
	 * 获取接口访问凭证
	 * 
	 * @param appid 凭证
	 * @param appsecret 密钥
	 * @return
	 */
	public static Object[] getAccessToken(String appid, String appsecret) {
		Object[] token = null;
		String requestUrl = TOKEN_URL.replace("APPID", appid).replace("APPSECRET", appsecret);
		// 发起GET请求获取凭证
		net.sf.json.JSONObject jsonObject = net.sf.json.JSONObject.fromObject(httpsRequest(requestUrl, "GET", null));

		if (null != jsonObject) {
			try {
				token =new Object[]{jsonObject.getString("access_token"),jsonObject.getInt("expires_in")};
			} catch (net.sf.json.JSONException e) {
				token = null;
				// 获取token失败
				log.error("获取token失败 errcode:"+jsonObject.getInt("errcode")+" errmsg:"+jsonObject.getString("errmsg"));
			}
		}
		return token;
	}
	/**
	 * 
	 * @Title: sendTemplateMessage
	 * @Description: 发送微信模板消息
	 * @param appid
	 * @param app_secrect
	 * @param jsonStr	模板消息规定的json字符串内容
	 * @return
	 */
    public static String sendTemplateMessage(String appid,String app_secrect,String jsonStr) {
    	//获取access_token
        Object[] accessToken= getAccessToken(appid, app_secrect);
        //获取请求路径
        String sendMsgCustomUrl = SEND_MSG_TEMPLATE_URL.replace("ACCESS_TOKEN", accessToken[0]+"");
        System.out.println(sendMsgCustomUrl);
        System.out.println(jsonStr);
        String result = httpsRequest(sendMsgCustomUrl, "POST", jsonStr);
        System.out.println(result);
        log.info("模板消息发送结果："+result);
        return result;
    }
    
	static SimpleDateFormat sdf =new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    public static String sendWxMessage(String appid,String app_secrect,String openid,String templateId ,String title,String content,String remark,String url) throws Exception{
    	if(StringUtils.isEmpty(appid)){
    		throw new Exception("appid不能为空");
    	}
    	if(StringUtils.isEmpty(app_secrect)){
    		throw new Exception("app_secrect不能为空");
    	}
    	if(StringUtils.isEmpty(openid)){
    		throw new Exception("openid不能为空");
    	}
    	if(StringUtils.isEmpty(templateId)){
    		throw new Exception("templateId不能为空");
    	}
    	if(url==null){
    		url="";
    	}
    	if(remark==null){
    		remark="";
    	}
		String nowDate=sdf.format(new Date());
    	String jsonStr="{'data':{'remark':{'color':'#000000','value':'"+remark+"'},'keyword1':{'color':'#000000','value':'"+nowDate+"'},'keyword2':{'color':'#000000','value':'"+content+"'},'first':{'color':'#000000','value':'"+title+"'}},'template_id':'"+templateId+"','topcolor':'#000000','touser':'"+openid+"','url':'"+url+"'}";
		jsonStr=jsonStr.replace("'", "\"");
		String result=sendTemplateMessage(appid, app_secrect, jsonStr);
		System.out.println(result);
		return result;
    }
}
