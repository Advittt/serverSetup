server:

```cs
using System;
using System.Collections.Generic;
using System.Net;
using System.Net.Sockets;
using System.Text;
using ServerData;
using System.IO;
using System.Threading;

namespace Server
{
    class Server
    {

        static Socket listnerSocket;
        static List<ClientData> _clients;
        public string id;



        static void Main(string[] args)
        {

            Console.WriteLine("Starting server on " + Class1.GetIPAdress());

            string sklCode = Class1.GetSchoolCode();
            Console.WriteLine("School code: " + sklCode);
            SendSklCode(sklCode);

            listnerSocket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
            _clients = new List<ClientData>();

            IPEndPoint ip = new IPEndPoint(IPAddress.Parse(Class1.GetIPAdress()), 4242);
            listnerSocket.Bind(ip);

            Thread listenThread = new Thread(ListenThread);
            listenThread.Start();

        }
        public static string SendSklCode(string sklCode)
        {
            return sklCode;
        }


        static void ListenThread()
        {
            for (; ; )
            {
                listnerSocket.Listen(0);
                _clients.Add(new ClientData(listnerSocket.Accept()));
            }

        }


        public static void Data_IN(object cSocket)
        {
            Socket clientSocket = (Socket)cSocket;

            byte[] Buffer;
            int readBytes;

            for (; ; )
            {
                try
                {


                    Buffer = new byte[clientSocket.SendBufferSize];
                    readBytes = clientSocket.Receive(Buffer);

                    if (readBytes > 0)
                    {
                        Class1 class1 = new Class1(Buffer);  
                        DataManager(class1);

                    }
                }
                catch(SocketException ex)
                {
                    Console.WriteLine("a client has left the chat");
                    Console.ReadLine();
                }

            }

        }



        public static void DataManager(Class1 p)
        {
            switch (p.packetType)
            {
                case PacketType.Chat:
                    foreach (ClientData c in _clients)
                    
                        c.clientSocket.Send(p.toBytes());
                                       
                    break;
            }
        }
    
    
    
    }
    class ClientData
    {
        public Socket clientSocket;
        public Thread clientThread;
        public string id;

        public ClientData()
        {
            
            id = Guid.NewGuid().ToString();
            clientThread = new Thread(Server.Data_IN);
            clientThread.Start(clientSocket);
        }
        public ClientData(Socket clientSocket)
        {
            this.clientSocket = clientSocket;
            id = Guid.NewGuid().ToString();
            clientThread = new Thread(Server.Data_IN);
            clientThread.Start(clientSocket);
            SendRegistrationPacket();

        }

        public void SendRegistrationPacket()
        {
            Class1 p = new Class1(PacketType.Registration, "server");
            p.Gdata.Add(id);
            clientSocket.Send(p.toBytes());
        }
    }

}

```

client:

```cs
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Net.Sockets;
using System.Net;
using ServerData;
using System.IO;
using System.Threading;


namespace Client
{
    class Client
    {

        public static Socket master;
        public string sc;
        public static string id;
        

        static void Main(string[] args)
        {
            Console.Write("Enter school code " + Server );
            string sc = Console.ReadLine();
            while(sc != sklCode)
            {
                Console.WriteLine("Wrong input");
                Console.Write("Enter school code ");
                sc = Console.ReadLine();

            }
        
            A: Console.Clear();
            Console.Write("Enter host IP adress: ");
            string ip = Console.ReadLine();


            master = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
            IPEndPoint ipe = new IPEndPoint(IPAddress.Parse(ip), 4242);

            try
            {
                master.Connect(ipe);
            }
            catch
            {
                Console.WriteLine("Could not connect to host!");
                Thread.Sleep(1000);
                goto A;
            }

            Thread t = new Thread(Data_IN);
            t.Start();
            for(; ; )
            {
                Console.Write("::>");
                string input = Console.ReadLine();
                Class1 p = new Class1(PacketType.Chat, id);
                p.Gdata.Add(sc);
                p.Gdata.Add(input);
                master.Send(p.toBytes());
            }
            
            
            
        }

        static void Data_IN()
        {
            byte[] Buffer;
            int readBytes;
            for (; ; )
            {
                try
                {
                    
                    Buffer = new byte[master.SendBufferSize];
                    readBytes = master.Receive(Buffer);

                    if (readBytes > 0)
                    {
                        DataManager(new Class1(Buffer));
                    }
                }
                catch (SocketException ex)
                {
                    Console.WriteLine("the server has disconnected");
                    Console.ReadLine();
                    Environment.Exit(0);
                }
            }
        }
        static void DataManager(Class1 p)
        {
            switch (p.packetType)
            {
                case PacketType.Registration:
                    id = p.Gdata[0];
                    break;

                case PacketType.Chat:
                    ConsoleColor c = Console.ForegroundColor;
                    Console.ForegroundColor = ConsoleColor.Cyan;
                    Console.WriteLine(p.Gdata[0] + ": " + p.Gdata[1]);
                    Console.ForegroundColor = c;
                    break;

            }
        }
    }
}

```

class1(serverData):

```cs
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using ServerData;
using System.IO;
using System.Runtime.Serialization.Formatters.Binary;
using System.Net;

namespace ServerData
{
    
    [Serializable]
    public class Class1
    {
        public List<string> Gdata;
        public int packetInt;
        public bool packetBool;
        public string senderID;
        public PacketType packetType;

        public Class1(PacketType type, string senderID)
        {
            Gdata = new List<string>();
            this.senderID = senderID;
            this.packetType = type;
        }
        public Class1(byte[] packetbytes)
        {
            BinaryFormatter bf = new BinaryFormatter();
            MemoryStream ms = new MemoryStream(packetbytes);

            Class1 p = (Class1)bf.Deserialize(ms);
            ms.Close();
            this.Gdata = p.Gdata;
            this.packetInt = p.packetInt;
            this.packetBool = p.packetBool;
            this.senderID = p.senderID;
            this.packetType = p.packetType;
        }


        public byte[] toBytes()
        {
            BinaryFormatter bf = new BinaryFormatter();
            MemoryStream ms = new MemoryStream();
            bf.Serialize(ms, this);
            byte[] bytes = ms.ToArray();
            ms.Close();
            return bytes;


        }





        public static string GetIPAdress()
        {
            IPAddress[] ips = Dns.GetHostAddresses(Dns.GetHostName());

            foreach (IPAddress i in ips)
            {
                if (i.AddressFamily == System.Net.Sockets.AddressFamily.InterNetwork)
                    return i.ToString();
            }
            return "127.0.0.1";

        }
        public static string GetSchoolCode()
        {


            var chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
            var stringChars = new char[5];
            var random = new Random();

            for (int i = 0; i < stringChars.Length; i++)
            {
                stringChars[i] = chars[random.Next(chars.Length)];
            }

            return new String(stringChars);


        }
       
    }

    public enum PacketType
    {
        Registration,
        Chat
        

    }
}

```
